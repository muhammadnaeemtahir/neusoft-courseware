# Chapter 0 Q&A: Engineering Basics & Course Environment

> Interview quick-prep: question + concise answer only. Deep dives (plain-language analogy, real failure cases, engineering reasoning, implementation walkthrough, alternatives) live in the detailed version [../qa-en/ch00-qa.md](../qa-en/ch00-qa.md).

## 5.1 TypeScript basics

**Q:** When CLI parsing fails, why return `{ ok: false, error }` instead of `throw`ing or returning `null`?

**A:** Returning a discriminated union **forces** the caller to handle the failure branch first — without checking `ok`, the compiler won't let you touch the success fields, and the code simply won't compile. Accidents like "forgot the null check and `undefined` spread everywhere" disappear at the root; error handling moves left to compile time. `throw` is beyond the compiler's reach (miss the catch and the exception rockets to the top), while `null` conflates "succeeded but no value" with "failure".

## 5.2 Async programming

**Q:** Can a Promise be cancelled? How is "cancellation" implemented in engineering? What is an AsyncIterable?

**A:** A Promise itself cannot be cancelled — once the order is placed there's no recall. In engineering, `AbortSignal` broadcasts the "stop doing this" signal, and the receiver checks it at suitable moments and stops on its own (**cooperative cancellation**); the request timeouts in Chapter 1 are built on it. An AsyncIterable is an async sequence that can be consumed item by item with `for await...of` — a stream where "goods arrive and get processed one by one"; Chapter 1's model reply of `AsyncIterable<LlmEvent>` is exactly this abstraction.

## 5.3 The Bun runtime

**Q:** Which things in the Node ecosystem does Bun replace with a single tool? Give one API this project uses for each.

**A:** Three-in-one: runtime + package manager + test runner. The runtime executes TypeScript directly (`bun src/index.ts` with no pre-compilation); the file/process APIs replace fs + child_process boilerplate (this project uses `Bun.spawn` to start subprocesses and `Bun.write` to write files); `bun test` is a built-in test runner, no need to install jest/vitest. Workspace is the mechanism for monorepos to share dependencies across packages; the OpenCode main repo uses it to organize packages/*, while this teaching repo is deliberately a single package.

## 5.4 CLI design

**Q:** Why are stdout and stderr two separate pipes? What is the three-tier convention for exit codes?

**A:** stdout is the **protocol channel** (results consumed by humans or programs); stderr is the diagnostics channel (logs, errors). The pipe `|` only takes stdout — one stray log line mixed in and downstream JSON parsing crashes. Exit codes: 0 success, 1 runtime error (the task failed), 2 usage error (bad arguments); shell scripts branch on them, and "glance at whether the terminal shows error text" is unreliable. This project centralizes them into an `EXIT_CODES` constant, banning magic numbers.

## 5.5 Git workflow

**Q:** Both tag and branch point at commits — what's the difference? How do the three tiers of rollback rank by danger?

**A:** A branch moves forward with new commits; a tag is pinned forever — hence milestones get tags (this chapter: `v0.1-cli`), and `git diff v0.1-cli v0.2-provider` is the inter-module comparison. The three rollback tiers: `git restore <file>` discards working-tree changes (local); `git revert` creates a reverse commit (safe, keeps history); `git reset --hard` rewinds wholesale (dangerous, discards commits).

## 5.6 Test layering

**Q:** What do unit tests and integration tests each test? Why must the code under test do dependency injection?

**A:** Unit tests test pure functions, exhaustively enumerating inputs and outputs at millisecond speed (argument parsing, log formatting); integration tests test "assembled real behavior" — the CLI's value lives at the process boundary, so `Bun.spawn` starts a real subprocess and asserts the stdout/stderr/exitCode triple. Without injection, the code calls `new Date()` internally and opens IO directly — the test controls neither the clock nor the files, so results aren't reproducible; with injection, the test passes in a controllable fake clock and in-memory streams.

## 5.7 Structured logging

**Q:** Why must every log line carry a runId? Why must secrets never enter logs?

**A:** The runId is a trace ID — one per run, on every log line, so during an incident `grep <runId>` reproduces the full chain across files and processes. A secret (starting with `sk-`) that lands in a log file travels with the file — shared, backed up, uploaded — which equals a leak. So redaction happens **before serialization**, by field name — not "write it first, then delete it".

## 5.8 Project layering

**Q:** What does each of the four layers own? What is the one-sentence standard for judging "is the dependency direction right"?

**A:** domain (pure business rules and types, no IO) → application (orchestration, defining the flow of "one run") → adapter (translating the outside world into internal types, e.g. CLI, Provider) → infrastructure (SQLite, filesystem, Git). The standard: **the inner ring doesn't know the outer ring exists** — if a domain file imports an adapter, the direction is reversed, and that's an architectural smell.
