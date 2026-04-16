---
id: FEAT-002
title: "Orchestrator review-and-merge protocol"
type: feature
status: done
priority: high
created: 2026-04-16
updated: 2026-04-16
parent: EPIC-9cdd
dependencies: [FEAT-001, REFAC-001]
tags: [ticket-workflow, orchestrator, review-protocol]
agent_created: false
complexity: 6
---

# Orchestrator review-and-merge protocol

## Context

The orchestrator is the first automated quality gate between a sub-agent's
finished ticket and the epic branch it wants to merge into. Today the
workflow has no codified review/escalation protocol — orchestrators
improvise, and every downstream automation (the deferred PR review agent,
any future corrective-ticket path) would inherit whatever idiosyncratic
merge semantics the current session happens to produce.

This ticket lands the canonical protocol as a single cohesive change:
a reference document, an orchestration prompt template, and a new
`## Orchestrator Review Protocol` section in `SKILL.md` that points at
both. It depends on FEAT-001 (TASK template, so `CORRECTIVE_SUB_TICKET`
escalations have a file-shape contract) and REFAC-001 (complexity rubric,
so the reference's sizing example cites the calibrated rule rather than
the obsolete 4-files rule).

Source: `docs/implementation_plan.md` §2 "[FEAT] Orchestrator
review-and-merge protocol" (sequenced item #3). Primary source:
`docs/research/r2-orchestrator-review-protocol.md`. Resolves adversarial
review §2 (gate count / labeling, placeholder contradictions, grep-for-text
structure), §4 (escalation provenance, `_template.md` glob exclusion,
`MAX_FIX_CYCLES=2` flagged as extrapolation).

## Requirements

- [ ] Create `.claude/skills/ticket-workflow/references/orchestrator-review-protocol.md`
      with exactly **four** top-level `##` sections — no more, no fewer —
      named: `## Review Gates`, `## Escalation Actions`, `## Retry Budget`,
      `## Outcome States`.
- [ ] Create `.prompts/orchestration/_template.md` with the full
      `SYSTEM / ROLE … OUTPUT FORMAT` prompt reproduced verbatim from
      `docs/research/r2-orchestrator-review-protocol.md` lines 173–296
      (inside R2's `## Orchestrator prompt template` section). Prepend a
      3-line HTML comment that names the file's purpose, the
      `<MAX_FIX_CYCLES>` default (2), and a pointer to
      `.claude/skills/ticket-workflow/references/orchestrator-review-protocol.md`.
- [ ] Insert a new top-level section `## Orchestrator Review Protocol` in
      `.claude/skills/ticket-workflow/SKILL.md` **immediately after** the
      `## Epic Branch Workflow` section (and before `## Execution Protocol`).

### `## Review Gates` content

Reproduce R2 §"Orchestrator review checklist" items **1–12 verbatim** as
one numbered list. Every item's bold title must survive the copy —
including item 9 `**Search for "implicit behavior changes" in shared or
configuration surfaces.**` (this is the specific gate the adversarial
review flagged as commonly dropped). **Do not relabel as A–I.**

### `## Escalation Actions` content

The four actions from R2 §"Escalation pattern comparison" (R2's actual
heading — see `## Notes` below for the proposal's mis-attribution).
Present them with the following canonical action names, each mapped to the
failure-type rule lifted verbatim from R2:

1. `REASSIGN_NEW_SUB_AGENT` — from R2 §"Re-assign the same ticket to a new
   sub-agent".
2. `CORRECTIVE_SUB_TICKET` — from R2 §"Create a corrective sub-ticket
   targeting the specific failure". The corrective sub-ticket **must** use
   the TASK template landed in FEAT-001.
3. `ESCALATE_TO_PR_AGENT_WITH_FLAG` — from R2 §"Mark the ticket blocked
   and escalate to the downstream PR review agent with a flag".
4. `RESTART_FROM_SCRATCH` — from R2 §"Revert the sub-agent's branch and
   restart from scratch".

### `## Retry Budget` content

- `MAX_FIX_CYCLES = 2` as the default.
- One sentence noting: *"Extrapolated from R2, which leaves
  `<MAX_FIX_CYCLES>` as a placeholder. See `docs/implementation_plan.md`
  §4 for the observable signals that would justify changing this default."*

### `## Outcome States` content

The six terminal states, one per bullet:

- `MERGED`
- `NEEDS_FIX`
- `REASSIGNED`
- `ESCALATED`
- `RESTARTED`
- `BLOCKED_DEPENDENCY`

### `.prompts/orchestration/_template.md`

Reproducing R2 lines 173–296 preserves the full placeholder set R2 provides
(≥ 22 distinct tokens). The HTML-comment header must look like:

```
<!-- Orchestration prompt template. Do not edit per-epic instances here;
     this file is the authoring source copied to epic-<hex>_*.md files.
     <MAX_FIX_CYCLES> defaults to 2 (tunable per epic).
     See .claude/skills/ticket-workflow/references/orchestrator-review-protocol.md
     for the canonical protocol. -->
```

### `## Orchestrator Review Protocol` section in SKILL.md

Exactly:

- One paragraph pointing to both new files.
- One bullet listing the six outcome states in the exact form:
  `MERGED / NEEDS_FIX / REASSIGNED / ESCALATED / RESTARTED / BLOCKED_DEPENDENCY`.
- One sentence: *"The file `.prompts/orchestration/_template.md` is
  underscore-prefixed so it is **not** matched by the closure ticket's
  `epic-<hex>_*.md` glob; do not change that glob."*

## File path hints

- `.claude/skills/ticket-workflow/references/orchestrator-review-protocol.md`
  — create.
- `.prompts/orchestration/_template.md` — create.
- `.claude/skills/ticket-workflow/SKILL.md` — modify (insert new section
  between `## Epic Branch Workflow` and `## Execution Protocol`).

## Constraints

- Do NOT invent `<TICKET_PACKET>` or any placeholder token not present in
  R2 lines 173–296. Copy the prompt verbatim.
- Do NOT relabel R2's 12 checklist gates (no A–I re-lettering, no 1–9 with
  "omitted" gaps). All 12 survive.
- Do NOT rename the underscore prefix on `_template.md` — REFAC-002 and
  CHORE-001 both depend on the `epic-<hex>_*.md` closure glob excluding
  this file.
- Do NOT introduce any per-epic value (epic hex, branch name) into
  `_template.md` — those are placeholders populated at epic-creation time.
- Do NOT edit any R2 source file (`docs/research/r2-*.md` is a read-only
  research artifact).

## Acceptance criteria

- [ ] Protocol file structure exactly four top-level sections, correctly
      named:
      `[ -f .claude/skills/ticket-workflow/references/orchestrator-review-protocol.md ]`
      and
      `grep -cE '^## (Review Gates|Escalation Actions|Retry Budget|Outcome States)$'
      .claude/skills/ticket-workflow/references/orchestrator-review-protocol.md`
      returns exactly `4`.
- [ ] At least 12 numbered gates present in `## Review Gates`:
      `awk '/^## Review Gates$/,/^## Escalation Actions$/'
      .claude/skills/ticket-workflow/references/orchestrator-review-protocol.md
      | grep -cE '^[0-9]+\. \*\*'` returns ≥ 12.
- [ ] All four canonical escalation action names appear in `## Escalation
      Actions`: `grep -cE
      'REASSIGN_NEW_SUB_AGENT|CORRECTIVE_SUB_TICKET|ESCALATE_TO_PR_AGENT_WITH_FLAG|RESTART_FROM_SCRATCH'
      .claude/skills/ticket-workflow/references/orchestrator-review-protocol.md`
      returns ≥ 4.
- [ ] Prompt template exists and carries ≥ 6 of the verbatim R2
      placeholders: `[ -f .prompts/orchestration/_template.md ]` and
      `grep -cE '<(EPIC_NAME|EPIC_BRANCH|SANITY_COMMANDS|SIZE_THRESHOLDS|MAX_FIX_CYCLES|MERGE_METHOD)>'
      .prompts/orchestration/_template.md` returns ≥ 6.
- [ ] SKILL.md integration: exactly one new section header, the six-state
      bullet verbatim, and the glob-exclusion note all present:
      `grep -cE '^## Orchestrator Review Protocol$'
      .claude/skills/ticket-workflow/SKILL.md` returns `1`, **and**
      `grep -c 'MERGED / NEEDS_FIX / REASSIGNED / ESCALATED / RESTARTED / BLOCKED_DEPENDENCY'
      .claude/skills/ticket-workflow/SKILL.md` returns `1`, **and**
      `grep -c '_template.md'
      .claude/skills/ticket-workflow/SKILL.md` returns ≥ 1.

## Verification

```bash
PROTO=.claude/skills/ticket-workflow/references/orchestrator-review-protocol.md
PROMPT=.prompts/orchestration/_template.md
SKILL=.claude/skills/ticket-workflow/SKILL.md

# AC-1: four canonical top-level sections
[ -f "$PROTO" ]
[ "$(grep -cE '^## (Review Gates|Escalation Actions|Retry Budget|Outcome States)$' "$PROTO")" = "4" ]

# AC-2: ≥12 numbered bold-titled gates inside Review Gates
[ "$(awk '/^## Review Gates$/,/^## Escalation Actions$/' "$PROTO" | grep -cE '^[0-9]+\. \*\*')" -ge 12 ]

# AC-3: all four escalation action names present
[ "$(grep -cE 'REASSIGN_NEW_SUB_AGENT|CORRECTIVE_SUB_TICKET|ESCALATE_TO_PR_AGENT_WITH_FLAG|RESTART_FROM_SCRATCH' "$PROTO")" -ge 4 ]

# AC-4: prompt template exists with ≥6 R2 placeholders
[ -f "$PROMPT" ]
[ "$(grep -cE '<(EPIC_NAME|EPIC_BRANCH|SANITY_COMMANDS|SIZE_THRESHOLDS|MAX_FIX_CYCLES|MERGE_METHOD)>' "$PROMPT")" -ge 6 ]

# AC-5: SKILL.md integration
[ "$(grep -cE '^## Orchestrator Review Protocol$' "$SKILL")" = "1" ]
[ "$(grep -c 'MERGED / NEEDS_FIX / REASSIGNED / ESCALATED / RESTARTED / BLOCKED_DEPENDENCY' "$SKILL")" = "1" ]
[ "$(grep -c '_template.md' "$SKILL")" -ge 1 ]

# Sanity: new section lands between Epic Branch Workflow and Execution Protocol
python3 - <<'PY'
import re, sys
text = open(".claude/skills/ticket-workflow/SKILL.md").read()
idx_epic = text.find("## Epic Branch Workflow")
idx_orch = text.find("## Orchestrator Review Protocol")
idx_exec = text.find("## Execution Protocol")
assert idx_epic < idx_orch < idx_exec, (idx_epic, idx_orch, idx_exec)
PY
```

## Notes

- **Provenance correction.** `docs/implementation_plan.md` cites R2
  §"Escalation decision logic" as the source of the four actions. R2 has
  no such heading — the actions live under `## Escalation pattern
  comparison` (line 68). The plan's prose "correcting the proposal's
  provenance" is itself mis-named; use the actual R2 heading in the
  reproduced text and carry the canonical action names from the plan.
- **`MAX_FIX_CYCLES = 2`.** Extrapolated from R2 (which leaves the value
  as `<MAX_FIX_CYCLES>` placeholder). The `## Retry Budget` section must
  state this inline so the next adversarial cycle does not re-derive it
  from scratch. Observable signal for change is in plan §4.
- **Glob-exclusion rationale.** CHORE-001 will run
  `git rm .prompts/orchestration/epic-<hex>_*.md` at epic closure. The
  leading `_` on `_template.md` keeps it out of that glob — changing the
  prefix silently breaks closure cleanup across every epic.
- **Verbatim requirement.** R2 lines 173–296 include the
  `SYSTEM / ROLE`, `NON-NEGOTIABLES`, `INPUTS`, `REVIEW PROTOCOL`,
  `ESCALATION LOGIC`, and `OUTPUT FORMAT` blocks. Copy them exactly —
  paraphrasing at this stage voids the "single source of truth" contract
  the reference exists to establish.
