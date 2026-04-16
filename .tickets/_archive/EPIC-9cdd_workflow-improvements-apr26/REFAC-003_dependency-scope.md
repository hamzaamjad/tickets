---
id: REFAC-003
title: "Specify dependency ID resolution scope"
type: refactor
status: done
priority: medium
created: 2026-04-16
updated: 2026-04-16
parent: EPIC-9cdd
dependencies: [FEAT-001]
tags: [ticket-workflow, dependencies, templates]
agent_created: false
complexity: 2
---

# Specify dependency ID resolution scope

## Motivation

Bare IDs in a ticket's `dependencies` frontmatter field are ambiguous:
`FEAT-001` could refer to the same epic's `FEAT-001` or to an unrelated
epic's. The current SKILL.md does not say, and an agent that greps
globally is correct by one reading and wrong by another. Every ticket the
ticket-workflow skill itself produces uses bare IDs without friction —
which only works because every agent so far has adopted the same implicit
convention. This refactor writes the convention down.

Part 1 #3 of `docs/implementation_plan.md` additionally folds in the
adversarial review's fix for the Epic template (the Epic template has no
`dependencies:` field, so the annotation count must be derived from the
file rather than hard-coded).

Source: `docs/implementation_plan.md` §2 "[REFAC] `dependencies` ID
resolution scope". Resolves adversarial review §4 "off-by-one — Epic
template has no `dependencies:` field".

## Scope

Files with the code to refactor:

- `.claude/skills/ticket-workflow/SKILL.md` — the `## Naming` section
  (add a new subsection `### Dependency ID resolution`).
- `.claude/skills/ticket-workflow/references/templates.md` — the
  `dependencies: []` line in the `Feature`, `Bug`, `Chore`, and `Refactor`
  templates, plus the `Task` template (added by FEAT-001, which is this
  ticket's dependency). **The Epic template has no `dependencies:` field
  and is not touched.**

## Target structure

### New SKILL.md subsection `### Dependency ID resolution`

Insert under the `## Naming` section (two or three sentences, exactly):

> **Dependency ID resolution.** Bare IDs in a ticket's `dependencies`
> frontmatter field resolve within the ticket's current epic. Cross-epic
> dependencies must use the full form `EPIC-<hex>/TYPE-NNN`. An agent
> resolving a bare ID that has no in-epic match must fail the dependency
> check, not search other epics globally.

### templates.md: propagate the comment

For every template that has a `dependencies: []` line, replace the line
with:

```
dependencies: []         # bare IDs resolve within current epic; cross-epic: EPIC-<hex>/TYPE-NNN
```

This applies to `Feature`, `Bug`, `Chore`, `Refactor`, and — assuming
FEAT-001 has landed first per this ticket's declared dependency — `Task`.
FEAT-001's task template was authored with the annotation already in
place; this ticket does not need to edit the Task template if the
annotation already matches the target comment verbatim.

**Hard rule:** the Epic template must keep no `dependencies:` field. Do
not add one as part of this ticket.

## Acceptance criteria

- [ ] SKILL.md carries the resolution rule:
      `grep -c 'bare IDs resolve within current epic'
      .claude/skills/ticket-workflow/SKILL.md` returns ≥ 1.
- [ ] Every `dependencies: []` line in `templates.md` carries the full
      annotation, derived from the file itself (self-adjusting across
      FEAT-001's TASK template and the four pre-existing templates):
      `expected=$(grep -c '^dependencies: \[\]'
      .claude/skills/ticket-workflow/references/templates.md);
      actual=$(grep -c 'bare IDs resolve within current epic; cross-epic: EPIC-<hex>/TYPE-NNN'
      .claude/skills/ticket-workflow/references/templates.md);
      [ "$expected" = "$actual" ]` exits 0.
- [ ] Epic template remains unaffected — no `dependencies:` field
      introduced: `awk '/^## Epic$/,/^## /'
      .claude/skills/ticket-workflow/references/templates.md |
      grep -c 'dependencies:'` returns `0`.

## Constraints

- Do NOT introduce a `dependencies:` field in the Epic template. Epics
  declare their merge order in a separate `## Merge order` section and do
  not use frontmatter dependencies.
- Do NOT change the `dependencies` resolution contract itself (no
  fallback to global search, no wildcard syntax). Bare IDs scope to the
  epic; cross-epic uses the full form. That is the whole rule.
- Do NOT change the column width / indentation of the `dependencies: []`
  frontmatter line — templates align comments on a common column and
  other tickets grep-count them. Adjust trailing whitespace only.
- Pure refactor — zero behavior change outside what the new prose
  prescribes.

## Verification

```bash
SKILL=.claude/skills/ticket-workflow/SKILL.md
TEMPLATES=.claude/skills/ticket-workflow/references/templates.md

# AC-1: SKILL.md prose present
grep -q 'bare IDs resolve within current epic' "$SKILL"

# AC-2: annotation count matches dependency-field count (self-adjusting)
expected=$(grep -c '^dependencies: \[\]' "$TEMPLATES")
actual=$(grep -c 'bare IDs resolve within current epic; cross-epic: EPIC-<hex>/TYPE-NNN' "$TEMPLATES")
[ "$expected" = "$actual" ]

# AC-3: Epic template has no dependencies field
# (flag-form awk; the `/start/,/end/` form with both patterns matching `^## `
# captures only the start line itself.)
[ "$(awk '/^## Epic$/{flag=1;next} /^## /{flag=0} flag' "$TEMPLATES" | grep -c 'dependencies:')" = "0" ]
```

## Notes

- FEAT-001's TASK template is authored with the target annotation inline,
  so when FEAT-001 has landed first, REFAC-003's changes are limited to
  the four pre-existing templates (Feature, Bug, Chore, Refactor).
- If FEAT-001 is not yet landed at the time this ticket is picked up,
  block on the dependency — do not author a partial TASK template here.
