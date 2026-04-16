---
id: FEAT-004
title: "Worktree recovery runbook content (E1)"
type: feature
status: done
priority: medium
created: 2026-04-16
updated: 2026-04-16
parent: EPIC-9cdd
dependencies: [REFAC-001]
tags: [ticket-workflow, worktree, runbook, recovery]
agent_created: false
complexity: 5
---

# Worktree recovery runbook content (E1)

## Context

Git worktree failures in multi-agent flows — partial archives, mid-commit
crashes, `config.lock` contention, diverged epic branches — are rare in
the happy path and catastrophic in the edge cases. Today the ticket-workflow
skill has no recovery reference; an agent hitting a broken worktree
improvises, and improvisation around `git worktree prune`, `rm -rf` of
`.git/worktrees/`, or `git worktree repair` is how workspaces get
destroyed.

This ticket authors the runbook. A companion ticket (REFAC-005) wires it
into `SKILL.md` with two callouts and a serialization guard bullet. E1
stays focused on content so it fits under the complexity-tier threshold
landed in REFAC-001; if the runbook + SKILL.md integration were a single
ticket, the combined file count + scope change would push it across the
Tier-B threshold.

Source: `docs/implementation_plan.md` §2 "[FEAT] Worktree recovery runbook
content (E1)". Primary source: `docs/research/r5-worktree-failure-recovery.md`.
Resolves adversarial review §3 "runbook will not fit in a single ticket"
(split from SKILL.md integration), §3 "10-row table description undercounts
R5's 12" (specifies 12 rows), §3 "serialization rule unenforceable" (pairs
the rule with a retry+jitter shell snippet).

## Requirements

- [ ] Create `docs/runbooks/worktree-recovery.md` with three `##`
      top-level sections, in this order: `## Diagnostic commands`,
      `## Recovery procedures`, `## Prevention conventions`.
- [ ] `## Diagnostic commands` contains a 12-row table reproduced from
      R5 §"Quick-reference diagnostic table" (lines 29–44). Column headers
      must be `| Command | What it reveals | How to interpret output | Safety |`.
      The 12 rows (one command per row) are:
      1. `` `git worktree list --porcelain` ``
      2. `` `git worktree list --verbose` ``
      3. `` `git rev-parse --show-toplevel` ``
      4. `` `git branch --show-current` ``
      5. `` `git status --porcelain=v2 --branch` ``
      6. `` `git log --oneline -5 --decorate` ``
      7. `` `git diff --name-status` ``
      8. `` `git diff --cached --name-status` ``
      9. `` `git stash list` ``
      10. `` `git rev-parse --git-dir` ``
      11. `` `git rev-parse --git-path index` and `git rev-parse --git-path index.lock` ``
      12. `` `git worktree prune --dry-run --verbose` ``
      Each row carries R5's "what it reveals / interpret / safety" columns
      verbatim (citation markers may be stripped if desired).
- [ ] `## Recovery procedures` contains four `###` subsections, in this
      exact order and with these exact titles:
      1. `### Sub-agent killed mid-commit`
      2. `### Closure ticket partial execution`
      3. `### Worktree in inconsistent state`
      4. `### Epic branch significantly diverged from main`
      Each procedure reproduces R5's diagnose → stabilize → repair →
      verify sequence (R5 lines 50–380) and, where applicable, the
      Goal-A / Goal-B split for closure, plus the specific commands
      `git restore --staged --worktree :/`, `git worktree repair`, and
      `git worktree prune`.
- [ ] `## Prevention conventions` contains R5's six rules (R5 lines
      381–401). For the **second** rule (serialize `git worktree add`),
      embed the following verbatim shell snippet **below** the rule text
      so it becomes a mechanism, not just a policy:
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

## File path hints

- `docs/runbooks/worktree-recovery.md` — create. Parent directory
  `docs/runbooks/` must be created if missing (`mkdir -p`).
- `docs/research/r5-worktree-failure-recovery.md` — **read-only source**,
  do not edit.

## Constraints

- Do NOT add the SKILL.md callouts or the serialization bullet to
  `SKILL.md` — REFAC-005 owns those edits.
- Do NOT reduce the table to 10 rows or fewer; every one of R5's 12
  commands must be present.
- Do NOT rename the four recovery-procedure `###` subsections — REFAC-005's
  callouts reference them by exact title.
- Do NOT paraphrase the recovery commands (`git restore --staged`,
  `git worktree repair`, `git worktree prune`). Those exact invocations
  must survive the copy.
- Do NOT introduce automation (shell scripts the runbook invokes). It is
  prose + commands for a human-or-agent operator.

## Acceptance criteria

- [ ] File exists: `[ -f docs/runbooks/worktree-recovery.md ]` exits 0.
- [ ] Diagnostic table has ≥ 12 command rows:
      `awk '/^## Diagnostic commands$/,/^## Recovery procedures$/'
      docs/runbooks/worktree-recovery.md | grep -cE '^\| *\`git '` returns
      ≥ 12.
- [ ] Four recovery-procedure headings present, exact titles:
      `grep -cE '^### (Sub-agent killed mid-commit|Closure ticket partial execution|Worktree in inconsistent state|Epic branch significantly diverged from main)$'
      docs/runbooks/worktree-recovery.md` returns exactly `4`.
- [ ] Retry+jitter mechanism present:
      `grep -c 'config.lock' docs/runbooks/worktree-recovery.md` returns
      ≥ 1 **and** `grep -cE 'jitter|rand\(\)'
      docs/runbooks/worktree-recovery.md` returns ≥ 1.

## Verification

```bash
RUNBOOK=docs/runbooks/worktree-recovery.md

# AC-1
[ -f "$RUNBOOK" ]

# AC-2: ≥12 command rows in the diagnostic table
[ "$(awk '/^## Diagnostic commands$/,/^## Recovery procedures$/' "$RUNBOOK" | grep -cE '^\| *`git ')" -ge 12 ]

# AC-3: four recovery subsections, exact titles
[ "$(grep -cE '^### (Sub-agent killed mid-commit|Closure ticket partial execution|Worktree in inconsistent state|Epic branch significantly diverged from main)$' "$RUNBOOK")" = "4" ]

# AC-4: retry+jitter mechanism
grep -q 'config.lock' "$RUNBOOK"
grep -qE 'jitter|rand\(\)' "$RUNBOOK"

# Sanity: three top-level ## sections in the right order
python3 - <<'PY'
text = open("docs/runbooks/worktree-recovery.md").read()
i1 = text.find("## Diagnostic commands")
i2 = text.find("## Recovery procedures")
i3 = text.find("## Prevention conventions")
assert 0 <= i1 < i2 < i3, (i1, i2, i3)
PY

# Sanity: key recovery commands survive
grep -q 'git restore --staged --worktree :/' "$RUNBOOK"
grep -q 'git worktree repair' "$RUNBOOK"
grep -q 'git worktree prune' "$RUNBOOK"
```

## Notes

- REFAC-005 depends on this ticket's runbook existing and on the four
  recovery-procedure titles matching exactly. A title typo here breaks
  REFAC-005's callouts, which grep for the title substring.
- The plan's original `10-row table` language (proposal Part 2 E) was a
  miscount against R5's 12 rows; this ticket fixes that.
- The retry+jitter snippet is a pattern, not a library — the rule is
  serialization plus jittered backoff on `config.lock` contention, and
  the AC asserts on the mechanism's presence, not on the specific loop
  count or sleep math.
