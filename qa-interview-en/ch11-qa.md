# Chapter 11 Q&A: Extensions

> Interview quick-prep: question + concise answer only. Deep dives (plain-language analogy, real failure cases, engineering reasoning, implementation walkthrough, alternatives) live in the detailed version [../qa-en/ch11-qa.md](../qa-en/ch11-qa.md).

## 5.1 Four Extension Mechanisms

**Q:** What is the one-sentence criterion for distinguishing the four extension mechanisms? What does the form determine?

**A:** **What form the extension takes**: a Custom Tool is data (one JSON declaration suffices), MCP is a process (independent lifecycle; it crashes on its own and upgrades on its own), a Skill is a document (enters context only, never the execution path), a Plugin is code (runs inside the host process; most capable, highest trust requirement). The form determines every engineering decision that follows — failure modes, testing methods, defenses.

## 5.2 Capability Discovery

**Q:** How does each of the two discovery paths declare "I exist"? What engineering principle do they share?

**A:** Directory convention (Custom Tools go in `.opencode-lab/tools/*.json`, Skills go in `.opencode-lab/skills/*/SKILL.md`; reading the file's content tells you what it can do) vs. protocol handshake (MCP starts the server process and does initialize, then sends tools/list to ask the server — like exchanging business cards at a first collaboration to confirm each other's specs). Shared principle: **discovery failure degrades instead of crashing** — a missing directory is the normal state of "no extensions", and one bad file affects only itself.

## 5.3 JSON-RPC

**Q:** Why must the request id exist? What is a "notification"? How do protocol errors differ from business errors?

**A:** Firing two requests back-to-back on a shared pipe is legal, and the server may finish answering the second one first — without an id you can't tell whose answer you received; the spec requires requests to carry an incrementing id and responses to echo it back, and the client pairs by looking up the id in its table. A notification is **a message without an id**: a one-way event push ("initialization finished"), needing no receipt, saving a pointless empty-response round trip. Protocol errors go through the `error` field (e.g. -32601 Method not found — the method doesn't exist / parameters are illegal); business errors are when the request is legal but the tool execution failed (`result.isError: true`) — the client must **check two places**.

## 5.4 stdio Transport

**Q:** Why does stdio need no network authentication? How is a server crash perceived immediately?

**A:** Treat the child process's stdin/stdout as a two-way message pipe, and security reduces to one simple question — **can this process be started**: whoever can start it could already run that program directly, so there's no need to stack network authentication on top (a network transport must answer "who may connect to the port, and do we need a token"). Crash perception: the host holds the process's lifecycle, and a process exit shows up immediately as **the stdout stream closing** — far faster than a network transport's laggy heartbeat/timeout perception.

## 5.5 Skill Injection

**Q:** Why inject only the directory and not the full text? How do you run the numbers?

**A:** Full-text injection: 20 skills × 3000 characters each ≈ 60000 characters — ten times the 6000-character total budget for rule injection; you'd be stuffed before doing any work. Directory injection: 20 lines × about 60 characters ≈ 1200 characters, capped by `maxSkills = 20` on top, pinning total cost at the constant level. The full text is fetched on demand by the agent with the read tool.

## 5.6 Plugin Hooks and Error Isolation

**Q:** Why is error isolation the first principle of a plugin system? How is the isolator implemented? What can beforeTool do?

**A:** Plugins inject code into the runtime, and plugin code is untrusted — even code the user wrote themselves, whose third-party dependencies will throw just the same; no hook's error may affect the core execution path (watertight compartments: whichever compartment floods gets sealed off and logged as a warning, and the ship sails on). The isolator wraps every hook call in try/catch, demoting errors to warnings. beforeTool returning `{cancel: true, reason}` can intercept execution — the extension point for security plugins and audit policies; afterTool serves audit statistics, and onEvent subscribes to all runtime events.

## 5.7 Timeouts, Crashes, and Versions

**Q:** For the external process's three abnormal states, what are each one's signal and mechanism? Why is the protocol version check placed at the handshake stage?

**A:** ① Unresponsive (deadlocked; the process is alive but not answering) — there's no signal, so only time works: each request's timeout fires on schedule, rejects, and is removed from the pending table. ② Crash (the process exits) — the stdout closing triggers failAll: set a flag so later requests short-circuit, clear timers, and reject every pending request one by one. ③ Orderly shutdown — close() idempotently drains the pending table and kills the process. Versions go through the initialize handshake's protocolVersion: the benefit is that failure happens **early and explains itself** — the handshake stage reports "version incompatible"; dragged out to the tenth business call, an error over a format difference looks like mysticism.

## 5.8 Trust Model

**Q:** How do the four mechanisms rank by trust level, highest to lowest? What is the Custom Tool's core line of defense?

**A:** Code (Plugin, highest — arbitrary code inside the host process) > data+command (Custom Tool — parameters can be influenced by the model) > process (MCP — there's a protocol boundary, but external behavior is uncontrolled) > document (Skill, lowest — executes no code and can at most pollute context). Trust level determines defense: Plugins backstop with error isolation; the Custom Tool's core defense is **how parameters are passed** — parameters spliced into the command string are shell injection, so parameters go via environment variables plus an environment whitelist; MCP tools are uniformly tagged `risk: "medium"` so the permission layer defaults to ask.
