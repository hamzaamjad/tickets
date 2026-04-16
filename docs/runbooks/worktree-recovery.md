# Worktree recovery runbook

Canonical "break glass" reference for `git worktree` failures that show
up in multi-agent ticket-workflow pipelines. The happy-path is covered
in `.claude/skills/ticket-workflow/SKILL.md`; this runbook is what
agents follow when something went sideways — partial archives, mid-commit
crashes, `.git/config.lock` contention, diverged epic branches, stale
worktree registrations.

Sources: `docs/research/r5-worktree-failure-recovery.md`. Every
procedure here follows the same arc: **diagnose → stabilize → repair →
verify.** Destructive steps are flagged explicitly.

## Diagnostic commands

The diagnostics below are designed to be *read-only or dry-run first*,
and to answer: (1) which worktrees are registered vs missing, (2) what
branch/HEAD each worktree is really on, (3) whether a lock or
in-progress operation is blocking Git, and (4) whether any committed
work is at risk.

| Command | What it reveals | How to interpret output | Safety |
|---|---|---|---|
| `git worktree list --porcelain` | Canonical, script-stable list of worktrees with `worktree`, `HEAD`, `branch`, `detached`, `locked`, `prunable` fields. | Any entry marked `prunable` indicates Git believes the worktree linkage is broken (e.g., gitdir points to a non-existent location). Use this to identify "ghost" worktrees and which branch is "in use". | Safe to run in any state. |
| `git worktree list --verbose` | Same as above, plus human-readable reasons for `locked` / `prunable`. | If verbose shows `prunable: gitdir file points to non-existent location`, you're looking at stale metadata (often fixed by prune/repair). | Safe to run in any state. |
| `git rev-parse --show-toplevel` | Absolute path to the current worktree root. | Use this to verify an agent is operating in the correct worktree directory (and not accidentally in the primary clone). | Safe to run in any state. |
| `git branch --show-current` | Current branch name (empty if detached HEAD). | Empty output + `git status` showing detached implies you need to reattach by switching/creating the expected branch before continuing work. | Safe to run in any state. |
| `git status --porcelain=v2 --branch` | Machine-readable status + branch headers (ahead/behind), plus indicators of operations in progress. | Prefer this in automation because porcelain output is intended to be stable across versions/config. If status fails with "index.lock exists", treat as lockfile recovery. | Safe to run in any state (may error if a lock blocks writers). |
| `git log --oneline -5 --decorate` | Last 5 commits and what refs point to them. | Confirms whether a "closure ticket" commit exists, and whether you're inspecting the intended epic/ticket branch tip. | Safe to run in any state. |
| `git diff --name-status` | Unstaged changes. | If you see large unexpected diffs while you think you're "clean", you may be in the wrong worktree, or a partial closure moved files. | Safe to run in any state. |
| `git diff --cached --name-status` | Staged changes relative to `HEAD`. `--staged`/`--cached` shows what would be committed next. | In mid-commit failures, this tells you whether the index is partially staged and needs to be reset/unstaged before finishing. | Safe to run in any state. |
| `git stash list` | Existing stashes (potentially used by prior recovery attempts). | If a prior agent stashed as part of recovery, you may need to apply/pop it after fixing worktree registration. | Safe to run in any state. |
| `git rev-parse --git-dir` | The current worktree's *actual* `$GIT_DIR` (often inside `.git/worktrees/<name>` for linked worktrees). | Use this before touching any internal files; in worktrees, `.git` is a file and paths differ from expectation. | Safe to run in any state. |
| `git rev-parse --git-path index` and `git rev-parse --git-path index.lock` | Exact filesystem path to the active index and its lock file, respecting worktree relocation and env vars. | If `index.lock` exists and no Git process is running, it's usually safe to delete to unblock. | Safe to run in any state. |
| `git worktree prune --dry-run --verbose` | Shows what stale worktree metadata Git would remove, without removing anything. `--dry-run` (`-n`) is explicitly supported. | If dry-run lists the missing worktree you expected to remove anyway, you can proceed to the real prune during recovery. | Safe **only** as dry-run; real prune requires verification. |

## Recovery procedures

Each procedure follows a "diagnose → stabilize → repair → verify"
pattern. When a step is destructive, it is called out explicitly.

### Sub-agent killed mid-commit

**Symptom identification**

Run these inside the sub-agent worktree directory:

```bash
git rev-parse --show-toplevel
git status --porcelain=v2 --branch
git diff --cached --name-status
git diff --name-status
```

Typical symptoms include (a) `git status` or `git add` failing with an
`index.lock` message, (b) partially staged changes (non-empty
`git diff --cached`), or (c) detached HEAD / wrong branch.

**Recovery procedure (safe sequence)**

1. Identify and (if needed) terminate any stuck Git process (POSIX):

```bash
ps aux | grep -E '[g]it|[c]laude|[a]gent'
```

2. Locate the correct lockfile path for *this worktree* (do not guess
   the path):

```bash
git rev-parse --git-dir
git rev-parse --git-path index.lock
```

`--git-path` is specifically meant to resolve the correct internal path
in linked worktrees.

3. If the lock exists and no Git process is running, delete the
   lockfile:

```bash
rm -f "$(git rev-parse --git-path index.lock)"
```

Deleting an orphaned index lock is the standard unblock once you've
verified nothing is actively running.

4. Re-check status and staged state:

```bash
git status --porcelain=v2 --branch
git diff --cached --name-status
```

5. If the index looks "half-staged" (you don't trust what's staged),
   unstage everything back to `HEAD` without discarding working
   directory changes:

```bash
git restore --staged :/
```

`git restore --staged` restores index contents from `HEAD` by default.

6. If HEAD is detached (branch name empty), reattach to the expected
   ticket branch. Prefer switching to the existing branch; if it
   doesn't exist locally, create it at the current commit:

```bash
git branch --show-current
git switch "epic-<hex>/<ticket-id>/<slug>" 2>/dev/null || git switch -c "epic-<hex>/<ticket-id>/<slug>"
```

7. Restage and complete the commit:

```bash
git add -A
git commit -m "<ticket-id>: <message>"
```

**Verify clean state**

Run:

```bash
git status --porcelain=v2 --branch
test ! -e "$(git rev-parse --git-path index.lock)" && echo "OK: no index.lock"
git log --oneline -5 --decorate
```

If `index.lock` reappears immediately, suspect a tool repeatedly
spawning Git in the same repo (common in IDEs/automation); stop the
offending process before retry.

### Closure ticket partial execution

**Symptom identification**

In the orchestrator/epic worktree (expected branch `epic/<hex>/<slug>`),
run:

```bash
git branch --show-current
git status --porcelain=v2 --branch
git diff --cached --name-status
git diff --name-status
git log --oneline -10 --decorate
```

If you see lots of `renamed:` / `deleted:` / directory-wide changes,
you're likely mid-`git mv`/`git rm`. `git mv` updates the index
immediately but still requires a commit. `git rm` removes from the
working tree and index (unless `--cached`).

**Judgment call (cannot be fully automated)**

Choose one of these goals:

- **Goal A (recommended): abort and rerun closure cleanly** if the
  closure step is meant to be atomic and reproducible.
- **Goal B: finish the closure manually** only if (1) all intended
  operations are already reflected in the index/working tree and
  (2) you can validate the final layout confidently.

The safe default in automation is Goal A because it minimizes "mystery
state". This complements the idempotency guards in `SKILL.md` §Epic
Closure Ticket, which are designed so a clean rerun is a no-op on
already-completed steps.

**Recovery procedure (Goal A: abort closure and restore `HEAD`)**

1. Create a local safety branch at current `HEAD` (cheap rollback
   point):

```bash
git branch "backup/epic-<hex>-pre-closure-recovery"
```

2. Restore tracked files in both index and working tree back to `HEAD`
   (undo partial `git mv` / `git rm` impact on tracked paths):

```bash
git restore --staged --worktree :/
```

Restoring both index and worktree is explicitly supported by
`git restore`.

3. If closure created lots of *untracked* artifacts (e.g., temp
   directories) and they are blocking rerun, do a dry-run clean first:

```bash
git clean -n -d
```

Dry-run (`-n`) is best practice before any clean.

4. Only if the dry-run output is acceptable, perform the clean:

```bash
git clean -f -d
```

**This is destructive.** Real-world agent tooling has caused data loss
by running `git clean -fd` at the wrong scope, deleting untracked but
important files. Keep this step *narrow* and prefer not to run it in
the primary clone.

**Recovery procedure (Goal B: finish closure manually)**

If `git status` shows exactly the intended operations (archive move +
prompt removal + worktree artifact removal) and you want to complete:

1. Confirm staged content matches intent:

```bash
git diff --cached --name-status
```

2. Stage any missing tracked changes (only if needed), then commit:

```bash
git add -A
git commit -m "chore(epic-<hex>): archive + cleanup"
```

3. If the closure includes deleting tracked files, prefer `git rm` for
   them (not `/bin/rm`) so the index is consistent.

**Verify clean state**

```bash
git status --porcelain=v2 --branch
git log --oneline -5 --decorate
```

A clean closure ends with an empty status and a commit that contains
the archive/cleanup changes.

### Worktree in inconsistent state

This covers: directory missing but still registered; directory exists
but points to wrong gitdir; branch "in use" by a ghost worktree; or
metadata/directory mismatch from partial cleanup.

**Symptom identification**

From the *primary clone* (or any worktree), run:

```bash
git worktree list --porcelain
git worktree list --verbose
```

Look for `prunable` entries and unexpected `branch` fields.

**Recovery procedure (stale registration: directory missing)**

1. Dry-run prune to see what would be removed:

```bash
git worktree prune --dry-run --verbose
```

`prune` exists specifically to clean stale `$GIT_DIR/worktrees`
metadata.

2. If dry-run output matches the missing worktree(s), run the real
   prune:

```bash
git worktree prune --verbose
```

Git notes you can run `git worktree prune` in the main or any linked
worktree to clean stale admin files.

3. If you were blocked deleting a branch because it was "used by
   worktree", retry after pruning:

```bash
git branch -D "<branch-name>"
```

**Recovery procedure (directory exists but linkage is broken or moved)**

1. From the moved/broken worktree directory, attempt repair:

```bash
git worktree repair
```

Git explicitly documents `git worktree repair` for corrupted/outdated
worktree admin files and gives the moved-worktree and moved-main-repo
cases as examples.

2. If you moved multiple worktrees, repair can take paths:

```bash
git worktree repair "<path-to-worktree1>" "<path-to-worktree2>"
```

**Recovery procedure (branch is "in use" but you need it anyway)**

1. Identify which worktree claims the branch:

```bash
git worktree list --porcelain
```

2. If it's a real worktree, either remove that worktree (preferred) or
   detach it before reusing the branch. If you must check out a branch
   even though another worktree uses it, Git provides an override:

```bash
git checkout --ignore-other-worktrees "<branch>"
```

This is a "break glass" option and can confuse automation.

**Verify clean state**

From primary clone:

```bash
git worktree list --verbose
```

From the repaired worktree:

```bash
git rev-parse --show-toplevel
git branch --show-current
git status --porcelain=v2 --branch
```

### Epic branch significantly diverged from main

This scenario is about conflict recovery once all sub-ticket work is
already merged to the epic branch and you need a PR that merges
cleanly.

**Symptom identification**

In the epic/orchestrator worktree:

```bash
git branch --show-current
git fetch origin main
git log --oneline -5 --decorate
git log --oneline -5 --decorate origin/main
```

`git fetch` updates remote-tracking branches without touching your
working state, which is ideal before a risky integration.

**Recovery procedure (recommended for this workflow: merge main into
epic)**

In a multi-agent system, sub-agent branches often base off the epic
branch; rewriting epic history late can ripple downstream. Since your
epic branch is effectively a shared integration branch for sub-agents,
prefer a merge.

1. Create a backup pointer before conflict work:

```bash
git branch "backup/epic-<hex>-pre-main-merge"
```

2. Merge `origin/main` into epic (no fast-forward is fine either way;
   conflict handling is the same):

```bash
git merge origin/main
```

(If conflicts occur, Git will stop for resolution; you can later abort
if needed.)

3. Resolve conflicts, then stage and complete merge:

```bash
git status
git add -A
git commit
```

4. Optional but high-leverage: enable reuse recorded resolution
   (rerere) before doing repeated merges/retries:

```bash
git config rerere.enabled true
```

Git documents rerere as recording and replaying conflict resolutions
once enabled.

5. If you need to abandon the merge attempt:

```bash
git merge --abort
```

Git documents `git merge --abort` and its limitations (it can fail to
fully reconstruct pre-merge local changes if the tree was dirty).

**Recovery procedure (only if epic is truly private and no downstream
depends on it: rebase)**

If you are *certain* no other branches/worktrees depend on the epic
branch tip (rare in multi-agent setups), you can rebase onto
`origin/main`, but expect conflict resolution per-commit and
potentially a force push later. Git explicitly frames upstream rebases
as harmful when others base work on them.

**Verify clean state**

```bash
git status --porcelain=v2 --branch
git log --oneline -10 --decorate
```

Then run your normal test/CI gate for the epic. (Tests are outside
Git's scope but should be non-negotiable once conflicts are resolved.)

## Prevention conventions

These are concrete rules drawn from `git worktree` documentation and
multi-agent practitioner reality.

1. **Never delete worktree directories "by hand" as the normal path.**
   Always prefer:

   ```bash
   git worktree remove <path>
   ```

   Git documents `remove` as the proper lifecycle end, and notes you'll
   otherwise rely on prune/GC to clean metadata. This alone prevents
   most "branch in use" ghosts.

2. **Serialize worktree creation and cleanup in the orchestrator.** If
   you launch multiple agents concurrently, *do not* let them run
   `git worktree add` simultaneously in the same repo. Real reports
   show `.git/config.lock` contention breaking parallel worktree
   creation without retry/backoff. The rule: one worktree creation at
   a time per repo, with retry+jitter on lock errors.

   ```bash
   # Serialize git worktree add; retry on .git/config.lock contention with jitter.
   for attempt in 1 2 3 4 5; do
     if git worktree add "$path" -b "$branch" "$base" 2>/tmp/werr; then
       break
     fi
     grep -q 'config.lock' /tmp/werr || { cat /tmp/werr >&2; exit 1; }
     sleep "$(awk -v s="$attempt" 'BEGIN{srand(); print s*0.2 + rand()*0.3}')"
   done
   ```

3. **Treat `git rev-parse --git-path …` as mandatory in automation**
   whenever you need to touch internal files (e.g., lockfiles). Git
   explicitly warns against assuming whether a path lives under
   `$GIT_DIR` vs `$GIT_COMMON_DIR` and recommends `--git-path` to get
   the final location, which is crucial in linked worktrees.

4. **Add a hard guard that prevents agents from writing to the
   orchestrator root / primary clone.** A practitioner pattern is to
   resolve real filesystem paths for both orchestrator root and
   worktree path at agent startup and fail if they match. This exists
   specifically to prevent "agent wrote to the wrong checkout" class
   failures.

5. **Prefer "restore/reset of tracked files" over "clean everything"**
   in recovery scripts. `git clean -fd` is a blunt instrument; it will
   delete *all* untracked files in scope, and real agent tooling has
   deleted unrelated, important untracked data by running it too
   broadly. The rule: if you must use `git clean`, always do
   `git clean -n -d` first, and constrain scope as narrowly as
   possible.

6. **Use `git worktree repair` and `git worktree lock` intentionally.**
   If you ever move worktrees (or the main repo) via external tooling,
   recovery is `git worktree repair` (not manual surgery). If you have
   worktrees on volumes that go missing, `git worktree lock` exists
   specifically to prevent their admin files from being pruned.
