# Chapter 8 Q&A: Context & Compaction

> Interview quick-prep: question + concise answer only. Deep dives (plain-language analogy, real failure cases, engineering reasoning, implementation walkthrough, alternatives) live in the detailed version [../qa-en/ch08-qa.md](../qa-en/ch08-qa.md).

## 5.1 Tokens and the context window

**Q:** Why estimate instead of using an exact tokenizer? What does "everything in one pot" mean? Which kind of over-limit outcome is more dangerous?

**A:** An exact tokenizer is a heavy dependency and per-token counting has overhead, while budget control only needs **monotonicity** — a longer text never estimates smaller, and cutting content always makes the estimate smaller (relative comparison is enough). With mixed Chinese-English text a single conversion rate distorts, so count by character class separately (ASCII ÷ 4 + everything else × 0.8). Everything in one pot: the system prompt, the full history, tool definitions, and the output space reserved for the model are all deducted from the same budget. Being **silently truncated** after exceeding the limit is more dangerous — the model keeps working in earnest on incomplete input.

## 5.2 Content priority

**Q:** When the budget is tight, who is sacrificed first? What is never given up? Which item is easily left out of the accounting?

**A:** The system prompt and project rules are never given up (they define "who you are and by what rules you work"); the first user message (the task goal) is never given up; the most recent round and the latest tool output keep their original text (the model is working on them); then the newer middle rounds; only last do you touch the original text of old tool outputs and drop the oldest middle rounds. In one sentence: **the closer content is to the current decision, the higher its priority; identity and goal outrank process**. The easily-missed item is **tool definitions** — each tool's name/description/schema easily exceeds a hundred tokens.

## 5.3 The sliding window

**Q:** What are the two fatal wounds of the bare "slide by message count" window? What is the fix? Why is the collection direction from newest to oldest?

**A:** (1) The cut may land exactly between an assistant's tool_call and its tool result — the toolCallId becomes an orphan and the Provider **rejects the whole request** (not a soft "the model is confused" problem, but a hard error that never even sends); (2) the task goal may be cut out — the model loses memory. The fix: upgrade the selection unit from "message" to "**round**" (a user message + the following assistant messages + all their tool results); with rounds as atoms, discarding keeps pairs naturally intact, plus two iron rules (keep the first-round goal, keep the last round). Collect from newest to oldest and stop when it won't fit — what remains is guaranteed to be the history "closest to the present".

## 5.4 The three treatments of tool output

**Q:** How do truncation, summarization, and referencing escalate in strength? Which fine-grained rule matters most?

**A:** Truncation (keep head and tail, drop the middle; the cheapest; fits logs where "the key information is at both ends") → summarization (condense into conclusions; fits outputs with semantic structure) → referencing (store the original outside the message and keep only a pointer; the most economical but adds one hop to retrieve). The teaching version implements truncation (head+tail + the "[旧工具输出已压缩]" sentinel marker, meaning "[old tool output was compressed]"). The most important fine rule: **the latest tool output always keeps its original text** — it is the information the model is consuming right now; truncating it directly causes wrong next decisions.

## 5.5 Compaction

**Q:** What is the essential difference between trim and compaction? Why does the summary use the user role instead of assistant? What if the LLM summarizer goes down?

**A:** Trim is **subtraction** (what's dropped is gone forever); compaction is an **equivalence transform** (old rounds are condensed into a summary that replaces them in place, with the information "mostly" still there — like distilling a two-hour meeting recording into a one-page memo). The summary uses the user role because a summary is not something the model said; it is **the runtime environment's retelling of history** — user is exactly the existing channel for "the environment speaking to the model"; using assistant would make the model cite the summary as "something I myself said", confusing retelling with memory. When the LLM summarizer throws, it automatically **degrades** to the built-in rule-based summarizer — compaction is routine on long tasks, and the session must not be sentenced to death over its failure.

## 5.6 Checkpoint and recovery

**Q:** What is the signal for judging recovery? Why is idempotency a "hard safety requirement" rather than icing on the cake?

**A:** The signal is **pending tool calls** — the assistant has declared a call but the tool_call row is still stuck in pending, meaning the process died right around execution; recovery = read out the complete prefix + judge where the break is + write the pending list and constraint wording into the recovery prompt. Idempotency is a hard safety requirement: pending may hold a shell command, and blindly retrying means **executing the dangerous action twice** — actions confirmed complete are never redone.

## 5.7 Adapting to different model context limits

**Q:** What does "parameterize, don't branch" mean? What does reserveOutputTokens guard against?

**A:** The budget is not if-else in code but a set of **data** (maxContextTokens, reserveOutputTokens) passed in per model — the authoritative source for the limit is the Provider's capabilities(). Parameters should scale with the model's size (smaller models compact earlier and keep less original text). The output reserve guards against "history crowding out output": the window isn't all for history — room must be left for the model's reply, like first marking off a block on the right side of the blackboard labeled "reserve for conclusions": no matter how full the front of the board gets, it can't encroach on that block, or there's nowhere to write when it's time for the answer.

## 5.8 Context quality assessment

**Q:** How do you turn "does the lost information matter" into observable, testable quantities?

**A:** Three observable quantities: the **removed list** (exactly which messages this trim dropped — auditable), **compactedRounds / beforeTokens / afterTokens** (compaction scale and compression ratio), and **survival assertions** — construct a long history containing the goal, file changes, and test conclusions in tests, then after compaction assert these key facts are still in the summary. This is the embryo of a "context quality regression test": baseline context quality the way you baseline performance; when you change the trimming strategy, run it once — red means you know what was lost.
