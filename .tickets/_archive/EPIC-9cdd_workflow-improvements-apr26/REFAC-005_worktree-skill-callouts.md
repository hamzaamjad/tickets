---
id: REFAC-005
title: "SKILL.md callouts + worktree serialization guard (E2)"
type: refactor
status: done
priority: medium
created: 2026-04-16
updated: 2026-04-16
parent: EPIC-9cdd
dependencies: [FEAT-004]
tags: [ticket-workflow, worktree, skill-md]
agent_created: false
complexity: 2
---

# SKILL.md callouts + worktree serialization guard (E2)

## Motivation

FEAT-004 authors the worktree-recovery runbook, but until SKILL.md points
at it, an agent hitting a broken worktree still has no discoverable
guidance — they only find the runbook if they already know the path. This
refactor adds two minimal callouts (at the top of `## Worktree Rules` and
at the top of `## Epic Closure Ticket`) and one bullet in `## Worktree
Rules > Key constraints` that turns the serialization-on-contention rule
from prose-only into an actionable pointer.

R5's "Documentation placement recommendation" (lines 403–414) explicitly
argues for this split: the recovery content lives in the runbook, the
happy-path doc (`SKILL.md`) gets two minimal callouts. This ticket is
that integration.

Source: `docs/implementation_plan.md` §2 "[REFAC] SKILL.md callouts +
worktree serialization guard (E2)". Depends on FEAT-004's runbook
existing with the referenced section titles intact.

## Scope

Files with the code to refactor:

- `.claude/skills/ticket-workflow/SKILL.md` — three surgical additions:
  two section-top callouts and one bullet inside the existing "Key
  constraints" bullet list of `## Worktree Rules`.

## Target structure

### Callout at top of `## Worktree Rules`

Inserted **immediately after** the section heading, before the current
opening paragraph:

> **Recovery:** If any `git worktree` step fails, stop and follow
> `docs/runbooks/worktree-recovery.md` § Diagnostic commands before
> retrying.

### Callout at top of `## Epic Closure Ticket`

Inserted **immediately after** the section heading, before the existing
"Every epic MUST include a final closure ticket..." paragraph:

> **Recovery:** If closure fails mid-run, follow
> `docs/runbooks/worktree-recovery.md` § Recovery procedures >
> Closure ticket partial execution (Goal A is the safe default).

### New bullet in `## Worktree Rules > Key constraints`

Append one bullet to the existing list (position: last or second-to-last,
alongside the other constraints):

> - **Serialize** `git worktree add` calls per repo. On `.git/config.lock`
>   contention, retry with jitter — see `worktree-recovery.md`
>   § Prevention conventions for the exact snippet.

The bullet **must** include the substring `.git/config.lock` (this is the
failure-mode pointer the plan's AC asserts on). The runbook reference in
the bullet uses the shorter `worktree-recovery.md` form so the
whole-file path-count stays at the two canonical callouts.

## Acceptance criteria

- [ ] Both full-path callouts present:
      `grep -c 'docs/runbooks/worktree-recovery.md'
      .claude/skills/ticket-workflow/SKILL.md` returns ≥ `2`.
- [ ] Serialization rule present:
      `grep -cE '(Serialize|serialize) +\`git worktree add\`'
      .claude/skills/ticket-workflow/SKILL.md` returns ≥ `1`.
- [ ] `.git/config.lock` mechanism pointer present:
      `grep -c '.git/config.lock' .claude/skills/ticket-workflow/SKILL.md`
      returns ≥ `1`.
- [ ] The two callouts attach to the correct sections (structural
      check): `awk '/^## Worktree Rules$/,/^## /' SKILL.md` contains one
      `**Recovery:**` line and `awk '/^## Epic Closure Ticket$/,/^## /'
      SKILL.md` contains one `**Recovery:**` line.

## Constraints

- Do NOT author the runbook content here — that belongs to FEAT-004.
- Do NOT change the `## Worktree Rules` or `## Epic Closure Ticket`
  section titles.
- Do NOT reorder or delete any existing bullets in `## Worktree Rules >
  Key constraints` — append only.
- Do NOT introduce a third top-level callout (one per relevant section;
  matches R5's placement recommendation).
- Pure refactor — zero behavior change aside from the added prose.

## Verification

```bash
SKILL=.claude/skills/ticket-workflow/SKILL.md

# AC-1: at least two runbook references
[ "$(grep -c 'docs/runbooks/worktree-recovery.md' "$SKILL")" -ge 2 ]

# AC-2: serialization rule with code-span around `git worktree add`
grep -qE '(Serialize|serialize) +`git worktree add`' "$SKILL"

# AC-3: .git/config.lock pointer present
grep -q '.git/config.lock' "$SKILL"

# AC-4: one **Recovery:** callout in each of the two sections
[ "$(awk '/^## Worktree Rules$/,/^## /' "$SKILL" | grep -c '^\*\*Recovery:\*\*')" = "1" ]
[ "$(awk '/^## Epic Closure Ticket$/,/^## /' "$SKILL" | grep -c '^\*\*Recovery:\*\*')" = "1" ]

# Sanity: the bullet sits inside Worktree Rules > Key constraints
awk '/^### Key constraints$/,/^## /' "$SKILL" | grep -qE '(Serialize|serialize) +`git worktree add`'
```

## Notes

- The implementation plan's AC #1 spells the expected runbook-reference
  count as `= 2`. That assumes the serialization bullet references the
  runbook by a shorter form (e.g., `worktree-recovery.md` §
  Prevention conventions, no leading `docs/runbooks/`). This ticket
  adopts that form deliberately so the two callouts remain the canonical
  full-path entries. If a future edit chooses to use the full path in
  the bullet too, the AC can be relaxed to `≥ 2` without loss of
  meaning.
- FEAT-004 must land first — the callouts' `§ Diagnostic commands`,
  `§ Recovery procedures > Closure ticket partial execution`, and
  `§ Prevention conventions` section names all depend on FEAT-004's
  runbook using those exact titles.
- The bullet's "see `worktree-recovery.md` § Prevention conventions for
  the exact snippet" is the pointer that makes the serialization rule
  enforceable: the snippet lives in the runbook, the SKILL.md bullet
  simply states the rule and directs readers to the mechanism.
