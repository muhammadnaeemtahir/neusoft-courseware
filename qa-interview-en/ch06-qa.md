# Chapter 6 Q&A: Permissions & Safety

> Interview quick-prep: question + concise answer only. Deep dives (plain-language analogy, real failure cases, engineering reasoning, implementation walkthrough, alternatives) live in the detailed version [../qa-en/ch06-qa.md](../qa-en/ch06-qa.md).

## 5.1 Least privilege and default deny

**Q:** Why doesn't pure default-deny work? The engineering answer is a "tiered default policy" — how exactly are the tiers divided?

**A:** Pure default-deny leaves the agent unable to move an inch — an agent that can't even read a file has no security, only unusability. Tiered defaults: **read-only and easily reversible** actions (reading files) are allowed by default; **side-effecting** actions (writing files, running commands) require human confirmation by default; **catastrophic** actions (recursive deletion, formatting a disk) are denied unconditionally. The companion principle is explainability: every decision must be able to answer "which rule, and why" (`EvaluatedPermission` carries `rule` and `reason` fields).

## 5.2 The allow / deny / ask three states

**Q:** What is the semantics of ask? What is the most common "corruption"? What is the iron rule?

**A:** ask is a **deferred decision**: the evaluation layer only says "needs confirmation" for now, and the final ruling is postponed to the interaction layer, where a human resolves it into allow or deny. The most common corruption is "treating ask as an alias for allow" — in scripted environments with nobody to answer, it quietly lets things through; the teaching edition's iron rule is the opposite: **unconfirmed = not allowed**. Every decision must carry its own `rule` (an enum, not a free-form string, so tests can assert the exact level hit) and `reason`.

## 5.3 Tool risk tiers

**Q:** How are the three tiers divided? Why is risk "tool metadata" rather than "a property of the request"? How are unknown tools handled?

**A:** low (read-only and easily reversible: read/glob/grep/echo/calculator), medium (side-effecting but recoverable: write/edit/apply_patch/test/task), high (arbitrary execution with unpredictable consequences: shell). Risk is **declared by a human at tool registration time** (`ToolDefinition.risk`), and the permission layer only looks it up per the declaration — the permission layer is decoupled from specific tools, so adding a tool requires no permission-code changes; the tiering is settled carefully once at registration, not improvised per request. Unknown tools are treated at the highest risk (fail-closed: with insufficient information, prefer to ask).

## 5.4 Path boundaries and symbolic links

**Q:** Why is the `startsWith(workspace)` prefix check wrong? What does each of the two checks look at?

**A:** The string `/ws + /../evil` still matches the prefix — the prefix check is effectively useless. The right approach has two layers: the **lexical check** (at the string level: resolve to an absolute path, compute the relation to the workspace via relative, and treat anything starting with `..` as out of bounds) plus the **filesystem check** (realpath resolves symbolic links into the real path and validates once more). The analogy: the lexical check reads the address hand-written on the shipping label, while realpath looks at which warehouse the package is actually delivered into — if the warehouse is out of town, no amount of tidy writing on the label helps. And the tool layer and permission layer form **defense in depth**: if one layer is bypassed, the other still intercepts.

## 5.5 The dangerous-command floor

**Q:** Why should `rm -rf /` not even be asked? What are the "floor" and "ceiling" metaphors? What are the regex's two failure directions?

**A:** Asking "allow?" is itself dereliction of duty — the answer must not depend on whether the user's finger slips, so a hit on a hard-coded pattern means deny: **configuration cannot loosen it, and neither can user confirmation**. The metaphor: the adjustable part of the permission system (user configuration) is the ceiling, raised or lowered as needed; the floor is the **safety lower bound** — it reads no configuration at all, so no configuration can step it down. The regex's failure directions: too loose gives false negatives (`sudo rm` bypasses), too tight gives false positives (`rm single.txt` gets blocked) — the trade-off is written into the tests: better a false positive that makes the model rephrase than a false negative that lets a disaster through.

## 5.6 Sub-agent permission inheritance

**Q:** What does "permission monotonic decrease" mean? Why a risk threshold instead of a tool-name blacklist?

**A:** What a sub-agent can do must be a **subset** of what the main agent can do: what the main lacks the sub definitely lacks, what the main has the sub may have less of, and the reverse is never allowed. The teaching edition implements a read-only role list plus a risk threshold (`risk !== "low"` means denied). No blacklist, because a blacklist inherently fails to guard against **newly added tools** (fail-open — anything not on the list passes, like a "no pets allowed" sign: everything not named by name can enter); a risk threshold automatically covers any non-read-only tool registered in the future (fail-closed, equivalent to a whitelist where only low holds a ticket to enter).

## 5.7 Approval granularity

**Q:** What are the three approval granularities? Which did the teaching edition implement? Why must the session-level cache key be tool+resource?

**A:** One-time allow (this call only), session-level allow (same tool + same resource isn't asked again within this session), permanent allow (persisted across sessions). The teaching edition implements only one-time: the simplest semantics, the most auditable. The session-level cache key must be the **combination of tool + resource** — remembering by tool name alone means "allowed to write a.txt" gets replayed onto "write b.txt", the old permission silently in force (like replaying an amusement-park wristband). The companion UX constraint: low risk passes straight through without pestering, and when asking, show reason and resource — blind-pressing y is a confirmation equivalent to no confirmation.

## 5.8 Audit logs and security testing

**Q:** What are the three engineering constraints on audit logs? The difference between "attack-style testing" and ordinary testing?

**A:** (1) **Redaction**: shell commands may embed tokens (`curl -H 'Authorization: Bearer sk-…'`); resource must pass redactSecrets before entering the ledger. (2) **Restraint**: record a resource excerpt, not full file contents, and truncate. (3) **Non-blocking**: auditing is an observation surface, not the critical path — a failed disk write degrades to an in-memory record and never interrupts the agent. Ordinary testing verifies "normal use works"; attack-style testing exhaustively enumerates bypass attempts from an **adversarial perspective**: dangerous commands, out-of-bounds paths, patch escapes, config privilege escalation, unknown tools. The append-only shape guarantees that three months later you can still reconstruct from the data "why this command was allowed to run back then".
