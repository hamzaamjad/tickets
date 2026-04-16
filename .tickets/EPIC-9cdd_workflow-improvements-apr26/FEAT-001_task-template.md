---
id: FEAT-001
title: "Add TASK template to references/templates.md"
type: feature
status: done
priority: high
created: 2026-04-16
updated: 2026-04-16
parent: EPIC-9cdd
dependencies: []
tags: [ticket-workflow, templates]
agent_created: false
complexity: 3
---

# Add TASK template to references/templates.md

## Context

Every downstream decomposition path in this epic emits `TASK` files:
`SKILL.md` Execution Protocol Step 4, the orchestrator's `CORRECTIVE_SUB_TICKET`
escalation (FEAT-002), and — when unblocked — the deferred PR-review-agent
ticket-materialization loop. Without a canonical `TASK` template, every
decomposer improvises, voiding the "structured, consistent work units"
contract every later improvement assumes. This ticket lands the template
first so the rest of the epic can cite it.

Source: `docs/implementation_plan.md` §2 "[FEAT] `TASK` template in
`references/templates.md`" (sequenced item #1). Resolves adversarial review
§4 "5-backtick fencing" and §4 "`dependencies: []` annotation" (folded in
here rather than duplicated in REFAC-003).

## Requirements

- [ ] Append a new `## Task` section to
      `.claude/skills/ticket-workflow/references/templates.md`, positioned
      **below** `## Refactor` and **above** `## Epic`.
- [ ] Wrap the template in a **5-backtick fenced block** (opener
      ` ```````markdown` ` and closer ` ```````  ` — matches every existing
      template in the file).
- [ ] Populate the frontmatter with exactly these keys (values shown are
      the literal comment annotations to include):
      - `id: TASK-XXX`
      - `title: ""`
      - `type: task`
      - `status: to-do`
      - `parent:                  # required — cannot be empty; TASKs only exist inside a parent epic`
      - `dependencies: []         # bare IDs resolve within current epic; cross-epic: EPIC-<hex>/TYPE-NNN`
      - `agent_created: true`
      - `complexity:              # 1-10, required for TASK; see references/complexity-scoring.md`
- [ ] Populate the body with exactly five `##` subsections **in this order**:
      `## Goal` (single-sentence statement), `## File path hints`,
      `## Constraints` (Do-NOT rules), `## Acceptance criteria` (≤3),
      `## Verification` (runnable commands).
- [ ] Do **not** include `Context`, `Description`, `Steps to reproduce`, or
      `Notes` sections. TASKs are decomposed in-session and do not stand
      alone.

## File path hints

- `.claude/skills/ticket-workflow/references/templates.md` — modify
  (append new `## Task` section between existing `## Refactor` and `## Epic`
  sections).

## Constraints

- Do NOT modify any existing template (`Feature`, `Bug`, `Chore`, `Refactor`,
  `Epic`). REFAC-003 owns the dependency-comment edit for those.
- Do NOT change the headings or ordering of subsections prescribed above.
- Do NOT include placeholder `complexity` as optional — for `TASK` the field
  is required.
- Must keep the 5-backtick fencing convention the file already uses.

## Acceptance criteria

- [ ] `grep -cn '^## Task$' .claude/skills/ticket-workflow/references/templates.md`
      returns `1`.
- [ ] `awk '/^## Task$/,/^## Epic$/' .claude/skills/ticket-workflow/references/templates.md | grep -E '^## (Goal|File path hints|Constraints|Acceptance criteria|Verification)$' | paste -sd, -`
      prints exactly:
      `## Goal,## File path hints,## Constraints,## Acceptance criteria,## Verification`.
- [ ] `awk '/^## Task$/,/^## Epic$/' .claude/skills/ticket-workflow/references/templates.md | grep -cE 'parent: *# required|dependencies: \[\] *# bare IDs'`
      returns `2` (both annotated fields present).
- [ ] Within the new `## Task` section, fences use 5 backticks:
      `awk '/^## Task$/,/^## Epic$/' .claude/skills/ticket-workflow/references/templates.md | grep -cE '^\`{5}'`
      returns exactly `2` (one opener line `` `````markdown ``, one closer line
      `` ````` ``).

## Verification

```bash
FILE=.claude/skills/ticket-workflow/references/templates.md

# AC-1: exactly one "## Task" heading
[ "$(grep -cn '^## Task$' "$FILE")" = "1" ]

# AC-2: five body subsections in the correct order and no others
awk '/^## Task$/,/^## Epic$/' "$FILE" \
  | grep -E '^## (Goal|File path hints|Constraints|Acceptance criteria|Verification)$' \
  | paste -sd, - \
  | grep -Fx '## Goal,## File path hints,## Constraints,## Acceptance criteria,## Verification'

# AC-3: both annotated frontmatter fields present
[ "$(awk '/^## Task$/,/^## Epic$/' "$FILE" \
     | grep -cE 'parent: *# required|dependencies: \[\] *# bare IDs')" -ge 2 ]

# AC-4: 5-backtick fencing inside the new Task section
[ "$(awk '/^## Task$/,/^## Epic$/' "$FILE" | grep -cE '^`{5}')" = "2" ]
```

## Notes

- The implementation plan's original AC #4 (`grep -cE '^\`{5}$' ... returns 14`)
  was based on a miscount: current file has 5 closing-only `````-exact lines
  and 5 `` `````markdown `` opener lines, so the whole-file `^\`{5}$` count is
  5, not 12, and the whole-file `^\`{5}` count is 10, not 12. The section-
  scoped `awk ... | grep -cE '^\`{5}'` check above preserves the plan's
  intent (verify 5-backtick wrapping of the new template) without depending
  on an incorrect baseline.
- The `complexity` comment `# 1-10, required for TASK` is tightened relative
  to the other templates; REFAC-001 harmonizes the comment across all
  templates once the complexity-scoring reference lands.
