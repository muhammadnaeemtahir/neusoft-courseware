# Chapter 10 Q&A: Multi-Agent

> Interview quick-prep: question + concise answer only. Deep dives (plain-language analogy, real failure cases, engineering reasoning, implementation walkthrough, alternatives) live in the detailed version [../qa-en/ch10-qa.md](../qa-en/ch10-qa.md).

## 5.1 Agent Role Definitions

**Q:** Which four dimensions does an agent role bundle together? Why do we say "a role is data, not a code branch"?

**A:** Working style (systemPrompt), available tools (allowedTools), permission ceiling (readOnly), step budget (maxSteps). The meaning of the declarative approach: adding a new role requires only **one new definition** — the delegation tool, the permission layer, and the loop all change by exactly zero lines. `name` is the join key running through all three layers — the permission layer matches the read-only list by name, the delegation layer validates the delegable set by name, and the session layer records the name into the child session.

## 5.2 Context Isolation

**Q:** Why can't a child agent see the parent conversation? What does the `Omit<...>` in the task tool's deps type condense? Why isn't compile-time blocking enough?

**A:** Three reasons for isolation: the parent context doesn't get stuffed full of investigation details (the parent receives only conclusions), the child task can be replayed and audited independently, and the model isn't led astray by the parent conversation's irrelevant history (the outside consultant gets only a one-page task brief). `Omit<AgentDeps, "provider"|"systemPrompt"|"allowedTools"|"maxSteps"|"permission">` removes five "role fields" — the constructor **cannot supply** these five things at the type level: "leaking them to the child isn't even possible." But a runtime spread doesn't respect Omit (`{...parentDeps}` still carries them through), so execute must also **spread first, then explicitly overwrite** the five fields — the type blocks compile-time carelessness; the overwrite blocks runtime leftovers.

## 5.3 Delegation and Result Return

**Q:** How is a child task's failure returned to the parent? What is a "self-healing error"?

**A:** Any non-completed status is reduced to a tool result with `ok: false`, and the error carries **two suggested ways out** for the parent model to choose from — the parent task is not interrupted. A self-healing error: the error message is written for the model to read, and comes with "what to do next"; after reading it, the model corrects itself in the next round — for example, when a role name is mistyped, the **delegable list** is placed directly into the error text, and the model retries in the next round with a legal name.

## 5.4 Parallel and Serial

**Q:** Why does the teaching version execute the tool calls within one round serially? What guarantees the "results return in call order" property of parallelism?

**A:** Serial buys two guarantees: tool message order = tool call order (the prerequisite for traceable protocol pairing), and writes naturally don't conflict (only one tool modifies a file at any moment). Parallelization uses `Promise.all` to launch calls simultaneously, and its **order-preserving property** guarantees the result array's order = the input array's order, independent of completion order — in `const [r1, r2] = await Promise.all([callA(), callB()])`, r1 is always A's result, even if B finishes first.

## 5.5 Child Agent Sessions

**Q:** What three values does parentSessionId deliver? How does "injecting persistence" leave the parent loop code unchanged by even one line?

**A:** Accountability tracing (who dispatched what, so problems can be located), independent history (investigation details live in the child session, replayable on their own, outside the parent context), and independent lifecycle (the child session's status follows the child task's terminal state without touching the parent). The mechanism chain: task.execute creates the child session → createSessionPersistence binds child.id → it is injected into the child runAgent — the append calls inside the child loop **don't need to know "which session I should write to"**; whatever is injected is what gets written.

## 5.6 The Three-Layer Permission Defense

**Q:** What does each of the three defense layers block? Why isn't layer 1 enough?

**A:** Layer 1, allowedTools: the tools list sent to the model is filtered by whitelist — write tools are **invisible** at the model level. Layer 2, readOnlyAgents: write tools are still registered in the shared registry (the model can hallucinate a call), and permission evaluation **denies** outright under the rule "a read-only role may not execute any non-low-risk tool" (the gate checks the card, not the person). Layer 3, the recursion ban: task is in no child agent's whitelist (a child can't spawn grandchildren). Defense in depth — if any layer is bypassed, the others still hold.

## 5.7 Child Task Failure and Timeout

**Q:** What runaway does each of the three timeout/budget layers block? Why isn't an overall wall clock implemented?

**A:** Tool-level (the registry's timeoutMs, default 30 seconds) blocks a single tool hanging; step-level (role maxSteps: explore 8 / plan 12 / build 24) blocks a child agent spinning forever; the overall wall clock blocks a whole delegation taking unbounded time — not implemented in the teaching version, which leaves the `deps.signal` injection as the extension point. The three abnormal endings (failed/aborted/max_steps) are uniformly reduced to `ok:false` and returned to the parent model.

## 5.8 Result Aggregation and Decisions

**Q:** When two child conclusions contradict each other, who arbitrates? What is code's role in aggregation?

**A:** **The final decision authority lives in the main agent's model**: child conclusions enter the parent context as tool messages, and the parent synthesizes the final output. Conflicts are not arbitrated in code — code only **returns faithfully**: if explore-A says version=1 and explore-B says version=2, both lines enter the context; the parent reads the file itself for one look and rules (cost: one file read), or delegates a third verification. The division of labor is also, simultaneously, the structural solution to concurrent-write conflicts: all writes funnel to the parent agent for serial execution, and every child is read-only.
