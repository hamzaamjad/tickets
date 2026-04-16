---
id: REFAC-002
title: "Remove docs/INDEX.md references from closure workflow"
type: refactor
status: done
priority: medium
created: 2026-04-16
updated: 2026-04-16
parent: EPIC-9cdd
dependencies: []
tags: [ticket-workflow, closure]
agent_created: false
complexity: 2
---

# Remove docs/INDEX.md references from closure workflow

## Motivation

The Epic Closure Ticket workflow references `docs/INDEX.md` as a
canonical index that the closure CHORE is expected to update. That index
does not exist in this workspace — no file named `docs/INDEX.md` is tracked
in `main`'s HEAD — and every closure run against the current SKILL.md
either (a) creates a phantom file nobody reads, or (b) must silently skip
its step 4 and hope the idempotency guard catches it. Either outcome
degrades the closure workflow.

The original proposal (`docs/implementation_plan.md` §2 "[REFAC] Remove
`docs/INDEX.md` from closure") removed the step from `SKILL.md` but left
two live `INDEX.md` mentions in the Epic template inside
`references/templates.md`. This ticket extends the change to both files so
neither emits a reference to a nonexistent index.

Source: `docs/implementation_plan.md` §2 "[REFAC] Remove `docs/INDEX.md`
from closure (SKILL.md + templates.md)". Resolves adversarial review §2
"Proposal #1 leaves `INDEX.md` references live in `templates.md`".

## Scope

Files with the code to refactor:

- `.claude/skills/ticket-workflow/SKILL.md` — the `## Epic Closure Ticket`
  section's numbered list (currently six items; must become five).
- `.claude/skills/ticket-workflow/references/templates.md` — the Epic
  template's `## Merge order` block (currently carries two `INDEX.md`
  mentions, one in the HTML comment and one in the final numbered bullet).

## Target structure

### SKILL.md `## Epic Closure Ticket`

Delete the current step 4 ("Update `docs/INDEX.md`") and renumber the
remaining five items so the list reads:

1. Mark all sub-tickets and epic as `done`.
2. Archive the epic folder.
3. Delete the orchestration prompt.
4. Clean up worktree artifacts.
5. Commit.

Leave each step's body text as-is aside from the renumber. Do not add a
new step and do not merge existing steps.

### templates.md Epic template `## Merge order`

Rewrite the HTML comment and the final numbered bullet to describe the
closure CHORE without any external-index reference. Target shape:

```
<!-- Add one row per sub-ticket. List sub-tickets in the order they should
     be merged to the epic branch. Note which can execute in parallel
     worktrees vs. must merge sequentially. The closure CHORE ticket must
     ALWAYS be last — it archives the epic and deletes the orchestration
     prompt. See SKILL.md "Epic Closure Ticket" for the full spec. -->

1. [TICKET-ID] ([rationale — why this merges first])
2. [TICKET-ID] ([rationale — dependency or conflict note])
N. CHORE-NNN — Epic closure (always last: archives epic, deletes orchestration prompt)
```

The exact wording of the rationale fragments is flexible; the hard
requirement is: **no substring `INDEX.md` survives anywhere in the file
after this edit.**

## Acceptance criteria

- [ ] Zero behavior change to any step body other than the deletion and
      renumber: the remaining five steps' content (mark-done text, archive
      `git mv` command, `git rm` command for the orchestration prompt,
      worktree cleanup `rm -rf`, commit message template) is unchanged.
- [ ] No `INDEX.md` reference survives in `SKILL.md`:
      `grep -c 'INDEX.md' .claude/skills/ticket-workflow/SKILL.md` returns `0`.
- [ ] No `INDEX.md` reference survives in `templates.md`:
      `grep -c 'INDEX.md' .claude/skills/ticket-workflow/references/templates.md`
      returns `0`.
- [ ] Closure list has exactly five top-level numbered items:
      `awk '/^## Epic Closure Ticket$/,/^## /'
      .claude/skills/ticket-workflow/SKILL.md | grep -cE '^[0-9]+\. \*\*'`
      returns `5`.

## Constraints

- Do NOT introduce a replacement index file. No `docs/INDEX.md` creation,
  no renaming to `docs/CATALOG.md` or similar.
- Do NOT rename the top-level section `## Epic Closure Ticket`.
- Do NOT collide with REFAC-004's renumber-by-name change — refer to
  closure steps by purpose (archive, prompt-deletion, worktree-cleanup)
  in any commit messages rather than by the numeric index that
  REFAC-004 replaces.
- Do NOT modify any other template (`Feature`, `Bug`, `Chore`, `Refactor`,
  `Task`). Only the Epic template's `## Merge order` block.

## Verification

```bash
SKILL=.claude/skills/ticket-workflow/SKILL.md
TEMPLATES=.claude/skills/ticket-workflow/references/templates.md

# AC-2 / AC-3: zero INDEX.md references in either file
[ "$(grep -c 'INDEX.md' "$SKILL")" = "0" ]
[ "$(grep -c 'INDEX.md' "$TEMPLATES")" = "0" ]

# AC-4: closure list has exactly five numbered items
# (flag-form awk; the literal `/start/,/end/` form in the plan would match
# its own start pattern on both endpoints and return just the header line.)
[ "$(awk '/^## Epic Closure Ticket$/{flag=1;next} /^## /{flag=0} flag' "$SKILL" | grep -cE '^[0-9]+\. \*\*')" = "5" ]

# Sanity: each of the five surviving action verbs is still present
grep -q "status: done" "$SKILL"
grep -q "git mv .tickets/EPIC-" "$SKILL"
grep -q "git rm .prompts/orchestration/epic-" "$SKILL"
grep -q "rm -rf .claude/worktrees/epic-" "$SKILL"
grep -q "archive epic and clean up orchestration artifacts" "$SKILL"
```

## Notes

- Parallel-lane coordination with REFAC-004 (idempotency guards) and
  REFAC-002 (this ticket): both renumber the closure list. REFAC-004's
  `## Acceptance criteria` reference the steps by **name** rather than by
  index precisely so the two can land in either order without colliding.
- If a future ticket genuinely needs a top-level initiative index, that
  is its own design exercise — don't re-introduce one here.
