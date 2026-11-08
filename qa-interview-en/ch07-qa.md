# Chapter 7 Q&A: Session & SQLite

> Interview quick-prep: question + concise answer only. Deep dives (plain-language analogy, real failure cases, engineering reasoning, implementation walkthrough, alternatives) live in the detailed version [../qa-en/ch07-qa.md](../qa-en/ch07-qa.md).

## 5.1 Relations among the five entities

**Q:** Why is tool_call pulled out of messages into its own table? Which table does each of its three lifecycle stages land in?

**A:** Because it has **a lifecycle of its own** (pending → success/error/denied/skipped) and **spans multiple messages**. Three stages: model-initiated — an assistant message row plus a tool_call row (status=pending); program execution — **not stored** (external side effects happen between the two transactions); result returned — a tool message row plus a rewrite of the tool_call status. The head and tail of one call live in two different messages, interlocking via the shared toolCallId. Events are a parallel append-only audit stream that never enters model requests.

## 5.2 Primary keys and indexes

**Q:** message uses a UUID while tool_call and project use natural keys — what is the reasoning for each? What famous trap does SQLite's foreign key have?

**A:** Messages have no naturally unique attribute, so they use `msg_`+UUID; tool_call takes the model-returned toolCallId as its primary key — a protocol-layer naturally unique identifier, and using the natural key incidentally buys **idempotency** (inserting the same toolCallId twice must collide on the primary key); project uses the SHA1 of the path — the same path always yields the same id (idempotent dedup). The trap: foreign key constraints are **off by default**; `PRAGMA foreign_keys = ON` is required before they actually take effect. Indexes are built for real queries (`listSessions` filtering by project needs idx_session_project).

## 5.3 Transactions and the two ledgers

**Q:** Why must the tool's execution itself never be wrapped in a database transaction? What are the "two ledgers"?

**A:** The database cannot roll back external side effects: once the tool has executed, the file is already on disk, and `ROLLBACK` cannot touch it. So what the database records is "**what happened**", not "the process of doing it" — one ledger for the world, one for the database. The correct sequence: first record the "declaration" (transaction one: assistant message + pending tool_call) → truly execute (touching the world's ledger) → then record the "result" (transaction two: tool message + status rewrite). The mechanism that dares guarantee this even through a power cut: WAL discards the whole span that never reached the COMMIT marker, and recovery starts from the last commit point — a "half transaction" simply cannot exist mechanically.

## 5.4 The repository layer

**Q:** What are the three motivations for a separate repository layer? What concretely does the "anti-corruption layer" do here?

**A:** (1) **Consistency rules centralized**: "message and call must share a transaction" is written once, and callers find it hard to violate even if they want to; (2) **testable**: tests construct the repository against an in-memory database without mocking the entire database; (3) **anti-corruption**: database dialects — SQL rows' snake_case, JSON strings inside text columns — never leave the repository boundary; mapping functions like toSession complete the translation at the door, and the business layer sees no trace of SQL. The Loop depends on the repository through an even narrower persistence interface and doesn't even import the repository's concrete types.

## 5.5 Migrations and version management

**Q:** What are the three properties of migrations? What real bug did the v3 migration fix? What is the analogy for a "composite primary key"?

**A:** **Ordered** (v2 may depend on tables built by v1), **idempotent** (judged by the "already applied" set in schema_migrations), **atomic** (each migration executes and registers inside a transaction). The real bug v3 fixed: the Mock's deterministic toolCallId collided on the global primary key across sessions — the second run in the same workspace failed outright with UNIQUE constraint failed; after changing to the (session_id, id) composite primary key, toolCallId only needs to be "unique within a session". The analogy: locating a residence in a housing compound uses "building number + door number" — the door number alone repeats all over the compound; only the two columns together are unique.

## 5.6 Event recording and projection

**Q:** Why does the event table only INSERT and never UPDATE/DELETE?

**A:** It is an **append-only event stream**: it records "what happened", separated from "current state" — the id uses AUTOINCREMENT to guarantee strictly increasing values, and the event sequence is itself a replayable stream of facts; projections are views computed from the fact stream. Rewriting history equals forging facts, so append-only is like a running account book: you may only keep writing line after line; tearing pages or scribbling corrections is forbidden.

## 5.7 Concurrency and idempotency

**Q:** How is message ordering guaranteed? What does WAL solve? Where does recovery's retry semantics come from?

**A:** Every message insert takes `MAX(seq)+1` **inside a transaction**, combined with `UNIQUE(session_id, seq)` — when two write paths race, the later committer hits the unique constraint and errors, rather than silently interleaving. WAL is SQLite's journaling mode: the default journal mode lets writes block reads, while WAL lets reads and writes proceed in parallel (writes are still serialized) — enough for a single-process agent. Idempotency: ensureProject uses "path SHA1, query before insert". The retry semantics come from the recovery protocol — after restart, **no blind replay**: first check the pending list to decide where to continue.

## 5.8 Export and deletion

**Q:** What is the biggest pitfall in deletion? How does the teaching edition delete? Which five categories of data must the export include?

**A:** The pit is **orphan rows**: the session is deleted, but message/tool_call/event still point at a nonexistent session (like a shipping label pointing at a pickup point that has already been emptied out). The defense: foreign-key ON DELETE CASCADE, or explicit in-order deletion at the application layer — the teaching edition chooses the latter and puts it **in one transaction** (deletion is also a consistency boundary: all five tables cleared or all preserved). The export is a complete snapshot: session, messages, toolCalls, events, permissionRequests — five categories fetched in one pass; with the export file in hand you can fully reconstruct what happened in the session.
