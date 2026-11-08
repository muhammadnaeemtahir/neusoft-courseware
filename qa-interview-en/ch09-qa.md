# Chapter 9 Q&A: Git Snapshot & Rollback

> Interview quick-prep: question + concise answer only. Deep dives (plain-language analogy, real failure cases, engineering reasoning, implementation walkthrough, alternatives) live in the detailed version [../qa-en/ch09-qa.md](../qa-en/ch09-qa.md).

## 5.1 Git's three trees

**Q:** What are the two `XY` columns of porcelain status? What do `??` and `M` mean respectively?

**A:** X is the state of the **index relative to HEAD**, Y is the state of the **working tree relative to the index**. `??` marks a new file present in neither HEAD nor the index (untracked); `M` marks a modification. All of the Agent's file modifications happen only at the working-tree level, so the standard answer to "what did the Agent change" = a porcelain status listing + a `git diff HEAD`.

## 5.2 Snapshot before edits, diff after

**Q:** What does the "baseline + delta" model mean? What primitive does the teaching version's snapshot use, and why not the production version's write-tree?

**A:** First fix the baseline (the pre-change snapshot, solving the time problem of "once overwritten, it's gone"); only then does every comparison have a reference point (the post-change diff, solving the visibility problem of "what changed"). The teaching version picks `git stash push --include-untracked`: stash is by nature the primitive of "push uncommitted changes onto a stack and return the working tree to clean", inherently **non-destructive** (content goes onto the stack rather than being deleted), and `--include-untracked` covers the new files stash ignores by default. The production version uses `git write-tree` to write a tree object hung on a dedicated ref — more precise but not dependent on stash semantics.

## 5.3 User changes vs Agent changes

**Q:** Why must the snapshot come before touching anything? What happens without one?

**A:** `git diff HEAD` is a whole-working-tree diff; it contains both the Agent's changes and the user's uncommitted changes that **already existed before the run**. The snapshot stores the user's uncommitted changes into the stash; only then is the Agent's diff purely the Agent's delta. Without a snapshot, the user's changes can be overwritten by the Agent's writes with **no way to recover**.

## 5.4 Diff display granularity

**Q:** What question does file-level diff answer, and what question does hunk-level diff answer?

**A:** File level (porcelain, one file per line) answers "**scope**" — 5 files changed, which added/modified/deleted; good for summary statistics and quick locating. Hunk level (unified diff hunks, the `@@ -l,n +l,n @@` line ranges) answers "**content**" — precise to the line; good for human review. Usually: file-level overview first, then drill down to hunk level on demand.

## 5.5 revert / restore / conflicts

**Q:** What are the semantic differences among the undo primitives? Why does the teaching version's restore pick `checkout stash@{N} -- .` and not delete the stash? Why is confirm a hard gate?

**A:** `git revert` generates a reverse commit (touches history, safe); `git checkout <ref> -- .` copies the ref's contents into the working tree (doesn't move HEAD, doesn't touch history — checkout wears two faces: switching branches and restoring paths); `git stash pop` retrieves and **deletes** the stack top; `git reset --hard` blasts back to a commit (discards all uncommitted content; banned). The teaching version uses checkout and doesn't delete the stash: if the overwrite was wrong, you can take from the stash once more (append-only keeps the retreat route open). confirm is a hard gate: restore is a **destructive operation** and requires explicit authorization.

## 5.6 Isolated workspaces and nested repos

**Q:** What real nested-repo incident did the teaching version step on? How is the guardrail built?

**A:** While smoke-testing inside a **subdirectory** of the host repo, the snapshot stashed away the uncommitted changes of the whole outer docs/ directory along with everything else — git commands locate by cwd but **act on the entire repository**. The guardrail: `gitRepoRoot` finds the repo root, and snapshotting is allowed only when the workspace is the repo root (`workspace-not-repo-root` refuses outright), preventing collateral damage to other people's uncommitted changes when running in a monorepo subdirectory.

## 5.7 Recovery strategy after test failures

**Q:** What are the three tiers of the graded playbook? Who owns the "restore decision", and why doesn't the Agent restore automatically?

**A:** (1) The task failed but the diff has value — preserve the scene for human review; (2) the changes are confirmed harmful — restore to the baseline (confirm as a hard gate); (3) the restore itself errors — restore once more from the stash (append-only guarantees this route is always open). The restore decision belongs to the **user/application layer**; the snapshot layer provides only primitives — "were these changes worth it" is a value judgment, and the Agent shouldn't restore automatically; at the same time, guard against the "mindlessly repeat restore+retry after failure" infinite loop.

## 5.8 The safety boundary for Git commands

**Q:** What's the difference between "API design blocking misuse" and "runtime interception"? What design philosophy does the confirm parameter embody?

**A:** Runtime interception: the wrapper provides every command (dangerous ones included) and relies on if-checks inside function bodies — wrong usage can still be written down and pass compilation; the defense line only lights up at runtime. API design blocking misuse: destructive commands (reset --hard, clean, push --force, filter-branch) **simply aren't in the wrapper** — there's nowhere to call them from. Writing confirm into the function signature (`snapshotRestore(workspace, snapshot, confirm)`) makes authorization part of the parameter table — every call site must make this decision explicitly, and "forgot to confirm" is a visible missing argument right at the call. Two more gates: the command-source gate (only safe commands hard-coded inside the wrapper are allowed; no dynamic assembly) and the resource gate (timeout + environment variable whitelist + stderr truncation).
