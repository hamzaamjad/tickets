---
id: REFAC-004
title: "Idempotency guards on closure steps (reference by name, not number)"
type: refactor
status: to-do
priority: medium
created: 2026-04-16
updated: 2026-04-16
parent: EPIC-9cdd
dependencies: []
tags: [ticket-workflow, closure, idempotency]
agent_created: false
complexity: 2
---

# Idempotency guards on closure steps (reference by name, not number)

## Motivation

The `## Epic Closure Ticket` section lists destructive commands — `git mv`
of the epic folder, `git rm` of the orchestration prompt, `rm -rf` of the
worktree directory — as unguarded invocations. A closure run that crashes
halfway through (network blip, sub-agent kill, branch conflict) leaves the
repository in a half-archived state, and re-running closure fails on the
first already-completed step. Every operator-grade runbook makes destructive
steps safe to re-run; SKILL.md does not yet.

Additionally, the adversarial review flagged that the original proposal's
fix referenced closure steps by numeric index ("step 4" etc.), which
collides with REFAC-002's renumber. Referring to steps by **name** — the
archive step, the prompt-deletion step, the worktree-cleanup step —
decouples the two refactors so they can land in either order without
colliding.

Source: `docs/implementation_plan.md` §2 "[REFAC] Idempotency guards on
closure steps (reference by name, not number)". Resolves adversarial
review §2 "step-numbering collision with #1".

## Scope

Files with the code to refactor:

- `.claude/skills/ticket-workflow/SKILL.md` — the `## Epic Closure Ticket`
  section's destructive commands.

## Target structure

### Preamble (top of section)

Insert one sentence **at the top** of `## Epic Closure Ticket`, before the
numbered list begins:

> Every step below must be safe to re-run; check existence before acting.

### Archive step

Replace the current bare `git mv` with the guarded form:

```bash
[ -d .tickets/_archive/EPIC-<hex>_<slug> ] || git mv .tickets/EPIC-<hex>_<slug> .tickets/_archive/EPIC-<hex>_<slug>
```

Intent: if the archive destination already exists, skip. Otherwise, move.

### Prompt-deletion step

Replace the current bare `git rm` with the guarded form:

```bash
[ -f .prompts/orchestration/epic-<hex>_*.md ] && git rm .prompts/orchestration/epic-<hex>_*.md || true
```

Intent: if the orchestration prompt file exists, delete it; otherwise no-op.

### Worktree-cleanup step

Replace the current bare `rm -rf` with the guarded form:

```bash
[ -d .claude/worktrees/epic-<hex> ] && rm -rf .claude/worktrees/epic-<hex> || true
```

Intent: if the worktree directory exists, remove it; otherwise no-op.

### Do NOT renumber by index

Other tickets that reference closure steps (REFAC-002's `## Merge order`
note, FEAT-004's runbook) must refer to them as "the archive step", "the
prompt-deletion step", "the worktree-cleanup step" — not as step 2, 3, 4.
This ticket does not renumber the list; any renumbering that happens in
parallel from REFAC-002 is absorbed without conflict because the names
identify the steps.

## Acceptance criteria

- [ ] Preamble sentence present:
      `grep -c 'safe to re-run' .claude/skills/ticket-workflow/SKILL.md`
      returns ≥ 1.
- [ ] Archive step guarded:
      `grep -c '\[ -d .tickets/_archive/EPIC-<hex>_<slug> \]'
      .claude/skills/ticket-workflow/SKILL.md` returns ≥ 1.
- [ ] Prompt-deletion step guarded:
      `grep -c '\[ -f .prompts/orchestration/epic-<hex>_\*.md \]'
      .claude/skills/ticket-workflow/SKILL.md` returns ≥ 1.
- [ ] Worktree-cleanup step guarded:
      `grep -c '\[ -d .claude/worktrees/epic-<hex> \]'
      .claude/skills/ticket-workflow/SKILL.md` returns ≥ 1.

## Constraints

- Do NOT change the order of the closure steps.
- Do NOT rename the section heading `## Epic Closure Ticket`.
- Do NOT refer to closure steps by numeric index anywhere in this edit
  (prevents collision with REFAC-002's renumber).
- Do NOT introduce a non-POSIX shell dependency (e.g., Bash 4-only
  constructs) — closure runs in the same shell as the rest of the
  workflow.
- Do NOT mask genuine failures: use `|| true` only on operations that
  are legitimately no-op when the target is absent. The archive step
  uses `||` guarding on existence, not on the `git mv` itself — a
  genuinely failing `git mv` must still fail.

## Verification

```bash
SKILL=.claude/skills/ticket-workflow/SKILL.md

# AC-1: preamble
grep -q 'safe to re-run' "$SKILL"

# AC-2: archive step guard (note the literal '<' and '>' in SKILL.md placeholders)
grep -qF '[ -d .tickets/_archive/EPIC-<hex>_<slug> ]' "$SKILL"

# AC-3: prompt-deletion step guard
grep -qF '[ -f .prompts/orchestration/epic-<hex>_*.md ]' "$SKILL"

# AC-4: worktree-cleanup step guard
grep -qF '[ -d .claude/worktrees/epic-<hex> ]' "$SKILL"

# Sanity: idempotency triggers on the "|| true" pattern for the two no-op-safe steps
grep -qE '\] && git rm .prompts/orchestration/epic-<hex>_\*.md \|\| true' "$SKILL"
grep -qE '\] && rm -rf .claude/worktrees/epic-<hex> \|\| true' "$SKILL"
```

## Notes

- Parallel-lane coordination with REFAC-002: REFAC-002 removes the
  `docs/INDEX.md` step and renumbers the list; this ticket refers to
  steps by name so the renumber is absorbed transparently. Merge order
  between the two is irrelevant.
- Bash glob expansion for the prompt-deletion guard assumes at most one
  match (the naming convention `epic-<hex>_<slug>.md` is unique per
  epic). If some future process produces multiple prompt files per epic,
  the guard must be reworked — but that is a different ticket.
- The plan's guard syntax `&& ... || true` swallows only the existence
  signal, not actual errors from `git rm` or `rm -rf`. Those surface
  through `set -e` in the closure context.
