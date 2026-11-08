# Chapter 13 Q&A: Integration & Delivery

> Interview quick-prep: question + concise answer only. Deep dives (plain-language analogy, real failure cases, engineering reasoning, implementation walkthrough, alternatives) live in the detailed version [../qa-en/ch13-qa.md](../qa-en/ch13-qa.md).

## 5.1 createApp, the composition root

**Q:** What is the essence of the integration problem? What kind of workshop does the "composition root" resemble? Which three benefits does the narrow interface bring?

**A:** The integration problem is not "missing features" but "**features that were never wired together**" (the config module reads out the permission mode, but nobody hands it to the permission module...). The composition root is like the car plant's **final assembly shop**: each shop builds its parts, and only final assembly bolts them into a complete car in order; the CLI and E2E are both "pick up the finished car" customers who never enter the shop to turn a wrench themselves. Three benefits of the narrow interface: assembly order is written in exactly one place (implicit sequencing does not get rewritten at multiple call sites); test injection points are concentrated (provider/ask/dbPath are the entire replacement surface); and module-composition correctness has a single acceptance checkpoint — both integration bugs in this chapter were fixed by touching only this one file.

## 5.2 The provider selection chain

**Q:** What is the order of the three-tier selection chain, and why? What does "the Mock is not an empty shell" mean?

**A:** Test injection > environment-variable key (real Provider, wrapped in withProviderRetry) > Mock demo script. The order embodies **determinism first**: explicit injection is the most trustworthy, env comes next, and the Mock as fallback makes "demonstrable with zero configuration" a hard promise. The Mock is not an empty shell: defaultDemoScript also runs the full two-turn "call tools → summarize" chain, so session list/show and the audit all have real content to look at under zero configuration. The CLI integration tests also explicitly blank the API key to force the Mock path, keeping tests immune to whatever is left over on the developer's machine.

## 5.3 The permission gate and the dual-write audit

**Q:** What are the two classic permission mistakes at integration time? Why does the audit need a dual write? What does the escape hatch change — and only what?

**A:** (1) The permission gate built as a global singleton — audit records cannot carry "which session produced this decision"; (2) the "allow everything" switch short-circuits the policy outright, tearing out the dangerous-command floor too. The right way: the permission gate is **built fresh on every run**, with the closure capturing the current session id; the audit is written to two places — the JSONL file (the cross-session running ledger, cat/grep-friendly at a glance) + the session table's permission_request (the official ledger that session show can export) — two copies backing each other up. The escape hatch (--allow-all) only flips config-level mode entries ask→allow; **it cannot touch the dangerous-command floor or the fail-closed default**. When there is no interactive channel, ask is treated as denial (unconfirmed = not allowed).

## 5.4 The nine-step integration scenario

**Q:** What are the five sequencing constraints? Give one counterexample each where the order is wrong.

**A:** (1) Project rules enter the system prompt before the first model call — counterexample: the fixture AGENTS.md says "修改后必须运行测试" (Chinese: "after making changes, you must run the tests"); if the rules arrive late, the model finishes editing the file and starts summarizing, and the tests never ran; (2) the session exists before the permission gate is built — counterexample: a gate built once globally cannot closure-capture this session's id, so the second session's decisions get booked under someone else's name (the bug this chapter's E2E genuinely caught); (3) compaction happens before request assembly — counterexample: trim first, fold later, old tool results eat the whole budget, recent messages get squeezed out of the window, and the model loses its memory; (4) the snapshot comes before the first write — counterexample: the baseline gets contaminated with files the Agent itself just wrote, and the diff under-reports; (5) the diff is computed once, after runAgent has fully finished — counterexample: changes made after the final turn are missing from the diff.

## 5.5 E2E and the chain of evidence

**Q:** What is the essential difference between E2E and unit tests? Why is "the only Mock is the model" a philosophy rather than just a technique?

**A:** The difference is **completeness of the chain of evidence**: a unit test verifies that some function returns the right value; E2E verifies that "every link the user can perceive actually happened" — the file really changed, the tests really ran, the permission was really granted, the diff is really readable. With every unit test green, E2E can still fail, because the mismatch only shows up at module boundaries. Wherever you draw the Mock boundary is as far as your proof reaches: mocking the model concedes that "how the model thinks" is uncontrollable; keeping the real filesystem, shell, bun test, Git, and SQLite declares that what you are proving is "what the Agent did" — all three pieces of physical evidence (readFileSync, git stash show, audit.jsonl) inspect the world outside the Mock.

## 5.6 Interrupted-run recovery

**Q:** Which mechanism delivers "no blind re-execution"? When the write already landed completely, what should the recovery turn do?

**A:** Mechanism = **persistence boundary + recovery prompt**: after a crash the database rests at the last complete boundary; on recovery you compute pendingToolCalls (the calls with no recorded result) and write them into continuationPrompt — explicitly saying "这些调用未执行完成，高风险动作必须重新走权限确认，不得自动重试" (Chinese: "these calls were not executed to completion; high-risk actions must go through permission confirmation again and must not be retried automatically"). When the write already landed completely (pending is empty), the recovery turn **should not** execute the same write again — the file content being unchanged is the testable evidence of "no re-execution."

## 5.7 The six test layers

**Q:** What question does each of the six test layers answer? When E2E fails, what is the right move?

**A:** Unit (is this pure function correct) → component (does this module agree with its direct dependencies) → integration (do the modules behave correctly assembled together) → security (can malicious input escalate privileges) → recovery (can we continue after a crash) → E2E (is the user's end-to-end experience correct). The point of layering is **localization speed and cost allocation**: when E2E fails, first see which layer's assumption it broke, go back to that layer and add a failing case, and only then let E2E turn green — instead of piling assertions into E2E. The pyramid shape: roughly 250 unit tests as the base, roughly 70 component/integration tests in the middle, and 3 E2E main lines on top.

## 5.8 The CLI command surface

**Q:** Why is the CLI the product's "user agreement"? How do streaming mode and `--json` mode divide the work?

**A:** Once shipped, the command surface (run/session/config + the options) is a promise to the outside — changing an argument or an exit code breaks users' scripts. At the process boundary, stdout remains the protocol channel: streaming mode prints the event stream straight to the terminal (the typewriter effect, for humans), and `--json` renders the same event stream as JSON Lines (for programs to consume) — one pipeline, two renderings; the protocol does not change with the output format.

## 5.9 Example projects and cross-language support

**Q:** Why must the example projects' target tests "start red"? The Go project has no Go toolchain installed on the teaching machine — where is its teaching value?

**A:** If everything starts green, the tests pass even though the Agent did nothing, and the demo proves nothing — the target tests must be pending implementation (/health and IsPalindrome start red), and only turning green after the Agent finishes forms a **closed loop**. The Go project is a carrier of "complete materials + constrained environment": the tool protocol (the read/edit/test parameters) carries no language assumptions, and the TS/Go difference collapses into a single command string on the test tool (bun test vs go test ./...); with no toolchain installed, the Agent receives a **structured error** of "command not found" — which is itself a demonstration of the design "tool failures also travel through the protocol, and the model can read and relay them."
