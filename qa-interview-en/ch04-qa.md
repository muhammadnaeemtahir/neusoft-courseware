# Chapter 4 Q&A: Agent Loop

> Interview quick-prep: question + concise answer only. Deep dives (plain-language analogy, real failure cases, engineering reasoning, implementation walkthrough, alternatives) live in the detailed version [../qa-en/ch04-qa.md](../qa-en/ch04-qa.md).

## 5.1 ReAct and the while loop

**Q:** Why must ReAct alternate "reasoning → action → observation"? Why does the engineering implementation use a while loop instead of recursion?

**A:** A single LLM call is one question, one answer — it cannot handle multi-step tasks like "edit code → run tests → edit again", because the model can't see the results of its own actions. The essence of ReAct is **feeding the observations of actions back into the conversation history**. Three reasons for the while loop: (1) the step cap and the cancellation check sit in one fixed spot every iteration passes through; (2) call-stack safety — each recursive "ask again" pushes a stack frame, a long task overflows the stack, whereas a while loop can turn ten thousand times with a constant stack depth; (3) all four terminal states return from the same function, so the control flow is obvious at a glance.

## 5.2 State machine and event-driven design

**Q:** What are the three acceptance criteria for the event stream? What property follows once they hold?

**A:** (1) `run_started` opens the run; (2) exactly one `run_*` terminal event closes it (one of, and only one of, completed/failed/aborted/max_steps); (3) every tool that actually executes in between appears in exactly the triple `tool_call_received → tool_started → tool_finished`. When all three hold, "**the event order is the control flow**": replaying the stream in time order re-enacts the whole run, tests compare against the expected sequence item by item, and any missing or extra event fails on the spot.

## 5.3 Turn boundary

**Q:** What does maxSteps actually count? How many tool calls can one model turn produce?

**A:** It counts **model turns** (the complete reduction of one stream request-response), not tool executions. One model turn may produce 0, 1, or several tool calls, but they all belong to the same turn and turns increments by only 1. Execution must wait until the whole stream ends and the tool call parameters are complete (fragments fully reassembled) — you cannot execute while still receiving.

## 5.4 Tool call ID pairing

**Q:** What is the provider protocol's hard constraint on the message chain? Why does "complete pairing = replayable"?

**A:** Every `tool_call` in an assistant message must later be answered by **exactly one** tool message with the same `toolCallId`, in the same order as the calls — missing, extra, or mismatched ids and the next request is rejected outright (the courier-tracking-number analogy: receipts must be filed under the same tracking number). Complete pairing = replayable: because the model service is **stateless**, every request resends the full history — as long as pairing is complete in the database, resending it as-is lets you keep going. That is the foundation of crash recovery.

## 5.5 The five classes of stopping conditions

**Q:** What do the four terminal states correspond to? Where do RunStatus and SessionStatus differ? What are the three design principles?

**A:** completed (a turn no longer requests tools), failed (provider error), aborted (cancellation), max_steps (steps exhausted). When persisted, SessionStatus has no max_steps — it maps to failed. Three principles: **completion belongs to the model** (the loop doesn't guess "is the task done"), **failure is layered** (only provider errors count as failure; a tool failure is a normal result), **boundedness comes from infrastructure** (maxSteps/cancellation/loop detection as the safety net).

## 5.6 Retry layering

**Q:** Why can the provider layer retry while the loop layer must never retry model requests?

**A:** A three-step argument: (1) the loop's error check comes before `messages.push`, so the failed turn's half-finished content hasn't entered history yet — at that moment the history is clean; (2) if the loop retried, there would be "a second version of the same turn's answer", but the first version's partial text was already forwarded via text_delta to the UI and persistence layer — the two versions don't match; (3) whether a retry is safe depends on whether the stream has already emitted content (the emitted flag), and only the provider layer itself has that information. So: at the provider layer, retry mechanically when zero content has been emitted; at the loop layer, failure is terminal; whether to rerun the whole task is left to the layer above.

## 5.7 Streaming output and persistence order

**Q:** What is the persistence step checklist? If the process crashes at any one step, what state is the database in?

**A:** user message → assistant turn (1 message + N tool call declarations) → each tool result → terminal state, one transaction per step, in an order consistent with the logic. A crash at any step leaves the database at the state of "**the last completed step**" — no half-written assistant message, and no "message present, tool call records lost". When the assistant turn is persisted, tool calls are first recorded as **pending** — that's the recovery signal in chapter 8.

## 5.8 Loop protection

**Q:** How is the signature for repeated-call detection computed? What happens when the threshold is reached?

**A:** Before each turn's tool loop a signature is computed (tool name + input JSON, e.g. `echo:{"message":"again"}`); identical to the previous turn means repeatCount+1, and reaching the threshold (3 in the demo) trips finishFailed **before any real execution** — the Mock fires the same call for 5 turns, but only 2 actually execute. This stops the model from burning money in a "call the same tool to do the same thing" death loop.

## 5.9 The task closed loop

**Q:** What are the four beats of the closed loop? Where is the loop's boundary of responsibility within it?

**A:** Goal (user prompt) → plan (the model forms a sequence of actions based on the system prompt) → execute (tool calls) → verify (run the tests and read the results); on verification failure, return to execution carrying the failure information. The loop's job is to make the ring **turn and finish turning**: all results are sent back, failures don't interrupt, and steps are bounded. The verify beat is prompt-driven (the build role's systemPrompt states "run tests after editing"); "plan first, then execute" is not hard-coded into the loop — planning strategy belongs to prompts and role definitions; the loop only handles mechanics.
