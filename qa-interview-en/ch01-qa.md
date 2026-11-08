# Chapter 1 Q&A: LLM Provider

> Interview quick-prep: question + concise answer only. Deep dives (plain-language analogy, real failure cases, engineering reasoning, implementation walkthrough, alternatives) live in the detailed version [../qa-en/ch01-qa.md](../qa-en/ch01-qa.md).

## 5.1 Chat Completions vs. the Responses style

**Q:** What is the essential difference between the two API styles? Why does the course pick Chat Completions?

**A:** Chat Completions is **stateless** — the server doesn't remember you, and every turn you must send the full conversation history back; the Responses style keeps conversation state on the server, and the client sends only increments. Why pick the former: it's the greatest common denominator — DeepSeek/GLM/Kimi/Ollama are all compatible — and mastering it lets you connect to most model services; switching to other formats later just means adding another adapter under the same `LlmProvider` interface, and the Agent core doesn't change a line. Detail: in streaming mode, token usage is not returned by default — you must explicitly pass `stream_options: { include_usage: true }`.

## 5.2 The four roles

**Q:** Why must a tool's execution result be its own separate message carrying `tool_call_id`, rather than being concatenated into the next user message?

**A:** Because one assistant turn can fire **multiple** tool calls, and each result must pair one-to-one with the call that initiated it — the model uses that id to know "which result belongs to which call". Concatenating into a user message loses the pairing relationship, and the Provider rejects it outright — it's a hard protocol constraint.

## 5.3 Tool definitions and function calls

**Q:** Is `arguments` in the response a JSON object or a string? Why?

**A:** A JSON **string**. Because in streaming mode the parameters arrive in fragments — you can only concatenate as a string and parse the whole thing once it's complete; an object can't be "half-concatenated". Also, the description is the model's main basis for choosing a tool — write it vaguely and the model will pick the wrong tool or invent parameters.

## 5.4 Auth, timeouts, and retries

**Q:** Why must network calls carry a timeout deadline? Which layer should retry logic live in?

**A:** One hung request can jam the entire Agent loop, so `AbortSignal.timeout(ms)` must auto-abort on schedule; the error names a timeout throws (TimeoutError/AbortError) are classified accordingly. Retries live in the **Provider's wrapper layer** (`withProviderRetry`), not the business layer; and only requests that "haven't sent any content downstream yet" may be retried.

## 5.5 SSE stream parsing

**Q:** What is the number-one source of SSE parsing bugs? What problem does `TextDecoder`'s `stream: true` solve?

**A:** The byte-boundary problem: what arrives over the network are byte chunks, and chunk boundaries have nothing to do with SSE event boundaries — or even with the boundary of a single multi-byte character — so the parser must buffer incrementally: `push` only emits complete events that have seen a blank line. One Chinese character is 3 UTF-8 bytes and can be cut in the middle by a chunk; `stream: true` lets the decoder remember "how many bytes are still missing" and, when the next chunk arrives, finish the character before outputting — no mojibake.

## 5.6 Adapter and Capability

**Q:** How is "swap the model without touching the core" achieved? What is Capability for?

**A:** The Agent core imports only internal types (`LlmRequest`/`LlmEvent`); vendor wire formats are translated both ways by the Adapter — like a power plug adapter: socket shapes differ from country to country, and the appliance only recognizes the unified plug. Capability is a Provider declaring what it supports (can it stream, can it do tool calls, what's the context cap); upper layers decide based on it (whether tools are usable, when to compress); start from the minimal three fields, and add more only when a real consumer appears.

## 5.7 Error classification and the retry invariant

**Q:** Why is retrying a 401 pointless, why does 429 need exponential backoff, and why are 5xx retryable? What is the most critical invariant of streaming retries?

**A:** Semantic differences: 401/403 is the wrong key — retry and it's still wrong; 429 is the server shouting "not so fast!" — retrying immediately pours fuel on the fire, and exponential backoff (`delay = min(initial * factor^(attempt-1), maxDelay)`) gives the server breathing room; 5xx is a transient fault, retryable with backoff; protocol errors gain nothing from retrying. The invariant: **once the stream has emitted any content event downstream, never retry wholesale** — otherwise the two attempts' text concatenates into "helhello" and is written into history, and every later turn the model sees the dirty text; the pollution never self-heals. The right move is to pass the failure through and let upper layers decide.

## 5.8 Mock and deterministic testing

**Q:** Why does the Mock Provider deliberately cut text and parameter JSON into small pieces before emitting?

**A:** So that the downstream "fragment reassembly" logic is **genuinely driven in every single test**. If the Mock only emitted complete objects, the reassembly code would never be executed by tests — the moment it meets real network fragments it crashes, while the tests stay all green. Also, when the Mock script runs out it returns an `exhausted` error, ruling out silent repetition masking the "wrong number of calls" bug; record-and-replay stores real SSE responses as test data and plays them back.
