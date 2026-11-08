# Chapter 3 Q&A: Coding Tools

> Interview quick-prep: question + concise answer only. Deep dives (plain-language analogy, real failure cases, engineering reasoning, implementation walkthrough, alternatives) live in the detailed version [../qa-en/ch03-qa.md](../qa-en/ch03-qa.md).

## 5.1 Path resolution and the workspace boundary

**Q:** What are the three classes of path attack surface? Which layer of checking stops each? Which normal situation is easily killed by mistake?

**A:** ① `../` relative-path escape (`src/../../etc/passwd`) — lexical check: after resolve expands, verify with relative; ② absolute paths writing outside directly — also a lexical check; ③ symlink swap (an `innocent.txt` inside the workspace symlinked to `/home/other/secret`) — lexical checks cannot see it at all; you must `realpath` to the true path and verify again. The fourth, easily mis-killed case: when writing a new file, the path does not exist yet and realpath fails — this is not an attack; it is the daily routine of write/apply_patch, and it must be distinguished from "exists but out of bounds."

## 5.2 Binaries and big files

**Q:** How does the read tool decide a file is binary? What does "a tool's error message is a navigation instruction" mean?

**A:** The NUL-byte heuristic — reading a NUL means it is probably not text; garbage carries no information and purely burns tokens. Big files (2MB cap) refuse full reads, and the error message steers the model to the two-step "grep to locate + offset/limit segments." That is the meta-principle: the error message tells the model "**what to do next**", not just "you are wrong."

## 5.3 Glob and Grep

**Q:** Why is result pagination not optional? The teaching build uses Bun.Glob and production uses ripgrep — what does this trade-off say?

**A:** Because results are unbounded — `**/*` on a real repository can return tens of thousands of paths, and a loose regex can match tens of thousands of lines. limit truncation + total-count hint + continue-query advice: none of the three can be missing. The trade-off: teaching implementations prioritize **comprehensibility** (in-process scanning, zero dependencies, fine for a few hundred files); production implementations prioritize **scale correctness** (ripgrep respects .gitignore, scans in parallel, performs at the million-file level).

## 5.4 Exact edits and patch

**Q:** Why does "fail rather than guess" actually have the lower expected cost? What is apply_patch's atomicity?

**A:** Fuzzy matching is the source of accidents: the model writes `old_string` from memory, off by one indentation, and fuzzy matching picks the "most similar" passage — often the same-looking code **inside another function** — the edit lands in the wrong place with tests still all green, and the error lurks until runtime. Exact matching failing at 0 occurrences means the model reads the true original next round and retries — **one certain failure exchanged for a guaranteed-correct edit**. apply_patch packs multiple operations into one patch; if any part fails validation, the whole patch takes no effect (atomicity).

## 5.5 The shell tool

**Q:** What are the four controlled-execution elements of the shell tool? What is `TERM: "dumb"` for?

**A:** ① stdout/stderr captured separately (success logs and error diagnostics take separate channels); ② the exit code is the sole authoritative success/failure signal (text shouting "success" does not count); ③ timeouts must kill the process (`sleep 30` must not hang the loop forever); ④ an environment-variable whitelist (the parent process's API key must not be inherited by child processes by default and leak). TERM=dumb makes the child believe the terminal does not support color, so it emits no ANSI escape sequences — those control characters are pure token noise to the model.

## 5.6 Concurrent writes and file locks

**Q:** What scenarios do atomic replacement, file locks, and workspace isolation each solve? Why doesn't the teaching build do locks?

**A:** Atomic replacement (write .tmp then rename) guards against "readers seeing a half-new-half-old hybrid"; file locks guard against "two processes read the balance → compute → write back, overwriting each other"; workspace isolation guards against "two Agents editing the same repo trampling each other's intermediate state." The teaching build not doing locks is an explicit scope decision: Agents call tools sequentially, single process, single workspace — the race window does not exist; it covers the core with something cheaper (mkdtemp test isolation + apply_patch two-phase disk writes). Understanding "what scale needs what level of concurrency control" matters more than locking blindly.

## 5.7 Toolifying LSP and the test runner

**Q:** What are an adapter's three duties when toolifying language infrastructure? Why does the test tool keep only the last 60 lines of output?

**A:** ① Input normalization (the model says "run the tests"; the adapter translates it into the underlying command); ② output normalization (tens of thousands of log lines compressed into a model-readable summary); ③ lifecycle management (the LSP server's startup, diagnostic push, and shutdown are all handled for the model). The test tool keeps only the tail 60 lines because failure summaries (failing case names, expect diffs, summary lines) almost always sit at the end of the output — "same execution core, different output policy."

## 5.8 Output compaction and context cost

**Q:** What failure does each of the three output-budget layers guard against? Why is it said that "compaction does not equal discarding information"?

**A:** Layer one, in-tool budgets (read's offset/limit, grep's limit) — without it the model devours most of the window in one call; layer two, continue-query hints ("continue with offset=N") — without it the model believes what it sees is everything and concludes from half the information; layer three, the Registry's backstop truncation (maxOutputChars) — against the accident of a tool author forgetting the budget. During compaction, keep metadata such as totals, original lengths, and scan statistics — the model needs them to plan the next step.
