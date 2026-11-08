# Chapter 12 Q&A: Effect vs Promise

> Interview quick-prep: question + concise answer only. Deep dives (plain-language analogy, real failure cases, engineering reasoning, implementation walkthrough, alternatives) live in the detailed version [../qa-en/ch12-qa.md](../qa-en/ch12-qa.md).

## 5.1 A lazy description of a program

**Q:** What question does each of `Effect<A, E, R>`'s three parameters answer? And the essential difference between Effect and Promise in one sentence?

**A:** A: what is produced on success? E: in which ways can it fail? R: which external services are required to run? Together they read like a **work order** — what gets delivered, how it can go wrong, and whose equipment you're borrowing, all printed on the form in black and white. The essential difference: **a Promise is eager** (the moment `fetch(url)` runs, the request has already been sent), **an Effect is lazy** (constructing an Effect merely "stores the how-to"; no IO happens until runPromise fires it).

## 5.2 Service and Context

**Q:** What is the pain of Promise-style dependency passing called? What does Context.Tag resemble?

**A:** **Signature contagion**: every function carries a deps parameter, threading from the entry point all the way to the deepest helper, and the ten layers in between that never touch the dependency are forced to carry it too. Effect's answer is **demand declaration**: the program says "I need a Registry" without caring where it comes from, and the runtime is responsible for honoring it. Context.Tag is like a **numbered locker ticket** — the number is unique across the whole building (the runtime addressing key), the ticket face is printed with the compartment's shape (compile-time reconciliation), and `yield* Tag` fetches the service.

## 5.3 Layer and runtime composition

**Q:** Which two problems does Layer solve? What does "build once, share everywhere" mean?

**A:** (1) Build order and the dependency graph are checked **by the type system** — service A's recipe declares "I need B," and a Layer composition missing B won't compile at all, so you never discover "config wasn't initialized" only at runtime; (2) **memoization** — expensive resources like a database connection pool are built once at runtime startup, and all consumers share that one instance instead of each call site doing its own `new`. Layer is like the canteen's **recipe card**: it spells out which ingredients it needs, which dishes it can serve, and how to cook; before meal service the kitchen stocks everything off the cards and cooks it once.

## 5.4 The failure channel

**Q:** What kind of boundary is `Effect.tryPromise`? Why does the shape of the error type matter?

**A:** It is the **translation boundary between Promise and Effect** — explicitly mapping the outside world's `unknown` exceptions into a controlled error type. Companion combinators: Effect.fail actively enters the failure channel, catchIf catches only failures matching a condition, recover converts a failure into a success value (the E channel drops to zero). Shape determines recovery power: a discriminated union (a `kind` field) lets every branch handle exactly one failure mode precisely, while a broad `Error` leaves you reading the message string and guessing.

## 5.5 Scope and resource management

**Q:** Which two blind spots does Promise's try/finally have? Why does Scope release in reverse order?

**A:** (1) Cancellation simply does not exist — an in-flight Promise cannot be told "stop doing that," and finally can only wait for it to finish naturally; (2) pairing logic rests on human discipline — one newly added early return can drop a cleanup. Effect uses **Scope**: resources register onto the scope, and on close they release in **reverse order of registration** — later-acquired resources may depend on earlier ones (open the database connection first, then open a cursor on it; closing must close the cursor first), and this holds for all three paths: success, failure, and interruption (the front-desk checkroom returns items in reverse ticket order).

## 5.6 Fiber, queues, and backpressure

**Q:** What does Fiber add over Promise.all? How does the backpressure mechanism work?

**A:** A Fiber is Effect's lightweight execution thread: fork an Effect and you get a Fiber, which you can **join** (wait for the result), **interrupt** (cancel), and observe the state of — concurrency becomes a composable, observable, steerable program instead of naked concurrency. Backpressure: when tasks arrive too fast, use a **bounded queue**; when the queue is full, "put" suspends — the upstream automatically slows down and memory stays under control.

## 5.7 Testing with Layers

**Q:** Which three classic mock pains do Effect-style tests eliminate? What is the yardstick for a "healthy mock"?

**A:** (1) Module-level mocks (jest.mock hijacks the loader — brittle and coupled to the implementation); (2) global variable replacement (`globalThis.fetch = ...` polluting each other); (3) hand-written test construction arguments whose signatures explode as dependencies grow. Layer/provide makes "production assembly" and "test assembly" two configuration files of the same mechanism: the program is written as usual, tests inject different service implementations, the compiler guarantees you did not miss an injection, and the program body does not change by one line. The yardstick: **mock only the boundaries (model, filesystem, network) — never mock the logic under test itself**.

## 5.8 The cost and payoff of migration

**Q:** What are the three yardsticks for adopting Effect? Why is this project's main line all-green without using Effect?

**A:** (1) Failure types need to propagate across many layers and be handled differently by different callers; (2) the same service needs swappable implementations (test/production/multiple backends); (3) resource lifecycles are complex enough that manual pairing starts causing incidents. If none of the three apply, Promise is more straightforward; two or more, and Effect starts paying back. The costs count too: a learning curve, E/R signatures infecting half the repo along call chains, and every boundary with an existing Promise library needing one wrapping pass. This project's main line uses Promise + explicit dependencies + result union types — at teaching scale that is entirely sufficient. Effect's payoff is **scale-dependent**, not correctness-dependent.

## 5.9 Avoiding over-abstraction

**Q:** What are the four signals of "over-abstraction"? How does this project demonstrate "restraint"?

**A:** (1) running on a single-layer call (no composition, so laziness is meaningless); (2) only one kind of error, handled on the spot (the E channel carries no information); (3) every dependency has the repo's only implementation and never needs replacing (the R channel yields no benefit); (4) pre-layering "for what might come later" (YAGNI). This project's restraint, demonstrated: the main line application.ts/loop.ts runs on Promise throughout, and Effect lives only in the `src/effect-lab/` comparison-experiment zone; compare.ts deliberately **keeps** the Promise version — it is the living proof that "when the business is simple, Promise is more straightforward," giving the migration decision a physical object to compare against. Another trap is the **demonstration effect**: seeing a production project use Effect and copying it wholesale, when production's problem scale (many services, long sessions, clustering) differs from your script's.
