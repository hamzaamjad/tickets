---
id: CHORE-001
title: "Epic closure: archive, delete orchestration prompt, clean worktree"
type: chore
status: to-do
priority: high
created: 2026-04-16
updated: 2026-04-16
parent: EPIC-9cdd
dependencies: [FEAT-001, REFAC-001, FEAT-002, REFAC-002, REFAC-003, REFAC-004, FEAT-003, FEAT-004, REFAC-005]
tags: [ticket-workflow, closure]
agent_created: false
complexity: 2
---

# Epic closure: archive, delete orchestration prompt, clean worktree

## Description

Final ticket in EPIC-9cdd. Executes on the epic branch
`epic/9cdd/workflow-improvements-apr26` before the PR to `main`, so that
`main` receives a clean state with the epic already archived and the
orchestration prompt removed.

This ticket follows the `## Epic Closure Ticket` protocol in
`.claude/skills/ticket-workflow/SKILL.md`. By the time this ticket runs,
both REFAC-002 (drop `docs/INDEX.md` step) and REFAC-004 (idempotency
guards + reference-by-name) will have landed on the epic branch, so this
ticket should follow the **post-refactor** five-step, name-referenced,
idempotent closure flow — not the pre-refactor variant that was active
at epic creation time.

## Tasks

- [ ] Mark all nine sub-tickets and the epic as `status: done` and update
      their `updated:` date. This ticket is one of the ten — mark it last,
      after the archive/delete/cleanup steps succeed.
- [ ] Archive step (idempotent): move the epic folder only if the archive
      destination does not already exist.
      ```bash
      [ -d .tickets/_archive/EPIC-9cdd_workflow-improvements-apr26 ] \
        || git mv .tickets/EPIC-9cdd_workflow-improvements-apr26 \
                  .tickets/_archive/EPIC-9cdd_workflow-improvements-apr26
      ```
- [ ] Prompt-deletion step (idempotent): delete the orchestration prompt
      only if it exists.
      ```bash
      [ -f .prompts/orchestration/epic-9cdd_workflow-improvements-apr26.md ] \
        && git rm .prompts/orchestration/epic-9cdd_workflow-improvements-apr26.md \
        || true
      ```
- [ ] Worktree-cleanup step (idempotent):
      ```bash
      [ -d .claude/worktrees/epic-9cdd ] && rm -rf .claude/worktrees/epic-9cdd || true
      ```
      Note: the active orchestrator worktree is *this* worktree; the
      skill's post-PR cleanup step removes it. During CHORE-001, only
      delete stale or sub-ticket-scoped worktrees under
      `.claude/worktrees/epic-9cdd/` if present — do not `rm -rf` the
      worktree you are currently standing in.
- [ ] Commit: single dedicated commit:
      ```
      EPIC-9cdd: archive epic and clean up orchestration artifacts
      ```

## File path hints

- `.tickets/EPIC-9cdd_workflow-improvements-apr26/` — move to
  `.tickets/_archive/EPIC-9cdd_workflow-improvements-apr26/`.
- `.prompts/orchestration/epic-9cdd_workflow-improvements-apr26.md` —
  delete (only this epic's prompt; never `_template.md`).
- `.claude/worktrees/epic-9cdd/` (and sub-worktree dirs) — remove per
  the existence-guarded command above.
- `.claude/skills/ticket-workflow/SKILL.md` § Epic Closure Ticket — the
  protocol this ticket follows.

## Constraints

- Do NOT delete `.prompts/orchestration/_template.md`. The glob
  `epic-9cdd_*.md` excludes it; the CHORE must follow this glob exactly.
- Do NOT introduce a `docs/INDEX.md` update step. REFAC-002 removed it.
- Do NOT amend or create any other ticket's content — all sub-ticket
  status changes are simple `status:` line edits and `updated:` date
  bumps in existing frontmatter.
- Do NOT merge this closure commit to `main` as part of this ticket.
  The PR from `epic/9cdd/workflow-improvements-apr26` → `main` is the
  next step after CHORE-001 commits, not part of CHORE-001.
- Do NOT stage files with `git add -A` or `git add .` — stage each
  moved/removed path by name.

## Acceptance criteria

- [ ] Every sub-ticket (FEAT-001 through REFAC-005, plus CHORE-001) has
      `status: done` in its frontmatter.
- [ ] Epic ticket `_epic.md` has `status: done` in its frontmatter.
- [ ] The epic directory lives at
      `.tickets/_archive/EPIC-9cdd_workflow-improvements-apr26/` and **not**
      at `.tickets/EPIC-9cdd_workflow-improvements-apr26/`.
- [ ] `.prompts/orchestration/epic-9cdd_workflow-improvements-apr26.md`
      no longer tracked: `git ls-files .prompts/orchestration/` contains
      the template (if present at `main` HEAD) but not the per-epic file.
- [ ] Exactly one closure commit on the epic branch, with message
      `EPIC-9cdd: archive epic and clean up orchestration artifacts`.

## Verification

```bash
# AC-1: all ten tickets have status: done
for f in .tickets/_archive/EPIC-9cdd_workflow-improvements-apr26/_epic.md \
         .tickets/_archive/EPIC-9cdd_workflow-improvements-apr26/FEAT-001_*.md \
         .tickets/_archive/EPIC-9cdd_workflow-improvements-apr26/FEAT-002_*.md \
         .tickets/_archive/EPIC-9cdd_workflow-improvements-apr26/FEAT-003_*.md \
         .tickets/_archive/EPIC-9cdd_workflow-improvements-apr26/FEAT-004_*.md \
         .tickets/_archive/EPIC-9cdd_workflow-improvements-apr26/REFAC-001_*.md \
         .tickets/_archive/EPIC-9cdd_workflow-improvements-apr26/REFAC-002_*.md \
         .tickets/_archive/EPIC-9cdd_workflow-improvements-apr26/REFAC-003_*.md \
         .tickets/_archive/EPIC-9cdd_workflow-improvements-apr26/REFAC-004_*.md \
         .tickets/_archive/EPIC-9cdd_workflow-improvements-apr26/REFAC-005_*.md \
         .tickets/_archive/EPIC-9cdd_workflow-improvements-apr26/CHORE-001_*.md; do
  grep -q '^status: done$' "$f"
done

# AC-3: epic is archived, not in active .tickets/
[ -d .tickets/_archive/EPIC-9cdd_workflow-improvements-apr26 ]
[ ! -d .tickets/EPIC-9cdd_workflow-improvements-apr26 ]

# AC-4: orchestration prompt removed; template preserved
! git ls-files .prompts/orchestration/ | grep -q '^.prompts/orchestration/epic-9cdd_'
# template presence depends on FEAT-002 landing first (if it did, the template lives under _template.md)

# AC-5: exactly one closure commit with the expected message
[ "$(git log --pretty=format:'%s' | grep -c '^EPIC-9cdd: archive epic and clean up orchestration artifacts$')" = "1" ]
```

## Notes

- This closure runs on the epic branch `epic/9cdd/workflow-improvements-apr26`.
  The post-PR worktree cleanup described at the bottom of SKILL.md
  `## Epic Closure Ticket` (removing `.claude/worktrees/epic-9cdd/*`
  after PR merge) is the orchestrator's job, not this CHORE's.
- All nine dependencies must be `status: done` before CHORE-001 starts.
  The dependency list is the execution-order contract the orchestrator
  enforces.
- If closure fails mid-run, follow `docs/runbooks/worktree-recovery.md`
  § Recovery procedures > Closure ticket partial execution (Goal A is
  the safe default). This runbook reference exists because FEAT-004 +
  REFAC-005 both land before CHORE-001.
