# Chapter 2 Q&A: Tool Registry

> Interview quick-prep: question + concise answer only. Deep dives (plain-language analogy, real failure cases, engineering reasoning, implementation walkthrough, alternatives) live in the detailed version [../qa-en/ch02-qa.md](../qa-en/ch02-qa.md).

## 5.1 The full lifecycle of a tool call

**Q:** Which five stages does one tool call cross? What is the failure-handling principle at each stage?

**A:** Registration (the developer hands it to the Registry) → Discovery (turned into the `tools` array for the model) → the model generates arguments → the runtime validates and executes → the result goes back. Failures each have their place: registration-stage duplicates or illegal names are developer bugs and should throw immediately; **apart from the registration stage, every failure converges into a ToolResult with `ok:false` returned to the model** — only by seeing structured errors can the model self-correct on the next round, and the Agent's multi-step capability is built on this path.

## 5.2 JSON Schema / Zod

**Q:** Why must you validate at runtime even with TypeScript types? What is the ecological niche of the three names?

**A:** Type erasure: TS types exist only at compile time, while model output is data "born" out of network JSON that the compiler has never seen — `{path:123}` can slip into a function declared `path: string`. Runtime validation is the only gate that stops it. Niches: JSON Schema is the "vocabulary standard" (using JSON to describe what JSON looks like), Ajv/Zod are ready-made wheels (validator implementations); the teaching build deliberately hand-writes a subset so every step is visible and testable.

## 5.3 Tool naming and descriptions

**Q:** Why are name/description "part of the prompt" rather than documentation? How does tool-argument design differ from API-argument design?

**A:** The model relies entirely on them to decide "when to use this tool, what arguments to pass" — a vague description directly causes wrong tool picks and wrong arguments, wasting a whole round on correction. API arguments are written for programmers (with IDE completion); tool arguments are written for a **probabilistic sampler** (it sees only text), so examples, enums, units, and bounds in the argument description beat type declarations. Names must be `[a-z0-9_]` and unique, validated immediately at registration (fail fast).

## 5.4 Registry and dependency injection

**Q:** Why factory functions + closures instead of a class? What does "assemble at startup, read-only at runtime" mean?

**A:** The Map, the listener array, and the call counter are all locked inside the closure; the outside can only reach them through the returned methods — state privacy is guaranteed by construction. Lifecycle: tools are all registered at application startup, after which the registry never changes; each `call` is just a table lookup plus execution. The Registry is at the same time the dependency-injection boundary — Loop, the permission layer, and the MCP adapter depend only on `registry.call` and `toModelTools`, so "swap in another set of tools" requires no changes to callers.

## 5.5 Validation-error reporting

**Q:** What are the three design points of the validation-failure report? Why is "splitting errors from values" better than try/catch?

**A:** ① Collect all errors at once (how many required fields are missing, how many types are wrong — say it all in one pass, reducing model round trips); ② errors face the model (with field names and allowed values, so the model can fix accordingly); ③ split errors from values — `ok:true` carries `value`, `ok:false` carries `errors[]`. Splitting beats try/catch: TS cannot force the caller to catch (miss it and the exception shoots to the top), whereas after splitting, the compiler won't let you take `value` before you check `ok`.

## 5.6 Result shapes

**Q:** What are the four levels of tool-result shape? Which level does the teaching build implement?

**A:** text (universal; what the model reads best) → JSON (keeps field boundaries; extraction is more precise) → file references (big results land on disk; only the path and a summary go back) → artifact (a product object carrying metadata). The teaching build implements the first two levels (`ToolResultContent = text | json`); the idea that "big output must not be stuffed wholesale back into context" is enforced starting from this chapter.

## 5.7 Idempotency, timeouts, cancellation, and max output

**Q:** What problem does the double-insurance timeout solve? Which real scenarios "replay" tool calls? Which tools are idempotent?

**A:** Naive racing is powerless against tools that ignore AbortSignal — like a trainee wearing headphones who never hears the whistle; only the patrolling guard (hardTimer) can forcibly clear the field when time is up. Three replay scenarios: ① crash recovery (the tool finished executing, the result was not persisted, then the process crashed); ② upper-layer retries; ③ the model sees `ok:false` and calls again with different arguments. Idempotency: calculator is naturally idempotent (`(1+2)*3` comes out 9 computed a hundred times), write/shell are not (side effects stack) — retry policy must be per-tool.

## 5.8 Audit events and trace

**Q:** Why not print tool calls directly with `console.log`? What does the observer pattern change here?

**A:** Three problems: tests cannot assert (output goes to stderr and is unreachable), formats are scattered (changing the format means N edits), consumers are welded in (switching to JSON Lines logging means editing the Registry body). After switching to event subscription: tests push listeners into an array and can precisely assert the event sequence, format is decided by subscribers, and the Registry knows nothing about its subscribers. In one sentence: **events are the protocol, logs are an implementation detail**. The four events `tool_call_requested → tool_started → tool_finished / tool_failed` carry a global callId, so any call can be stitched into a complete trace.
