# Orchestration prompt — EPIC-9cdd: Ticket workflow improvements (Apr 2026 synthesis)

> **Purpose.** Per-epic brief for the orchestrator agent responsible for
> executing, reviewing, and merging sub-tickets of EPIC-9cdd onto the
> epic branch `epic/9cdd/workflow-improvements-apr26` before the PR to
> `main`.
>
> **Note on bootstrapping.** This epic is the one that authors the
> canonical orchestration prompt template (`_template.md`) and the
> `references/orchestrator-review-protocol.md`. Until FEAT-002 lands,
> the orchestrator falls back on the minimal review discipline captured
> below. After FEAT-002 lands **on this epic branch**, the orchestrator
> should pick up the new template and protocol for every sub-ticket that
> follows.

## Epic facts

- **Epic ID / hex:** `EPIC-9cdd`
- **Epic branch:** `epic/9cdd/workflow-improvements-apr26`
- **Base branch:** `main`
- **Source plan:** `docs/implementation_plan.md` (the ten ticket-ready
  items + non-goals + open defaults)
- **Primary research:** `docs/research/r1-*.md` through `r5-*.md`
- **Goal:** Ship the Apr 2026 ticket-workflow improvements with every
  per-ticket `## Verification` shell check green on the epic branch.

## Merge order (authoritative)

Enforce strictly. Sequenced items (1–3) must merge in listed order.
Parallel-lane items (4–9) may merge in any order subject to their
declared `dependencies`. CHORE-001 always merges last.

| # | Ticket     | Depends on                                    | Notes |
|---|------------|-----------------------------------------------|-------|
| 1 | FEAT-001   | —                                             | TASK template. No deps. |
| 2 | REFAC-001  | —                                             | Complexity rubric + 2-tier policy. Parallel-safe with FEAT-001. |
| 3 | FEAT-002   | FEAT-001, REFAC-001                           | Orchestrator protocol + `_template.md`. Sequenced #3. |
| 4 | REFAC-002  | —                                             | Drop `docs/INDEX.md` from closure. Parallel. |
| 5 | REFAC-003  | FEAT-001                                      | Dependency-ID scope. Parallel, soft-deps on FEAT-001. |
| 6 | REFAC-004  | —                                             | Closure idempotency by-name. Parallel, coordinates with REFAC-002. |
| 7 | FEAT-003   | REFAC-001                                     | `## Outcome` schema + Lane A retrieval. |
| 8 | FEAT-004   | REFAC-001                                     | Worktree recovery runbook. Content only. |
| 9 | REFAC-005  | FEAT-004                                      | SKILL.md callouts + serialization bullet. |
| ω | CHORE-001  | ALL of the above                              | Closure: archive, delete prompt, clean worktree. |

## Epic-level constraints

Hard constraints for every sub-ticket merge decision:

1. **No changes outside the ticket's file path hints.** Out-of-scope
   edits are an automatic `CORRECTIVE_SUB_TICKET` or `REASSIGN_NEW_SUB_AGENT`
   (see "Review discipline" below for taxonomy).
2. **No `AGENTS.md` edits** (explicit non-goal, plan §3).
3. **No canonical-location symlink, no persistent vector index** (plan §3
   / §4 binding constraints).
4. **Verification-first.** Each ticket's `## Verification` shell block
   must be green before merge. "Tests passed" logs from the sub-agent
   are not sufficient; the orchestrator re-runs the block.
5. **TASK template inheritance.** Once FEAT-001 has merged, any
   decomposition-triggered sub-ticket created by the orchestrator or a
   sub-agent must use the TASK template from
   `.claude/skills/ticket-workflow/references/templates.md` § Task.
6. **Outcome append.** Once FEAT-003 has merged, every ticket marked
   `done` from that point on must carry an `## Outcome` section per
   `references/outcome-schema.md`, staged in the same commit as
   `status: done`.

## Review discipline (interim, pre-FEAT-002)

Minimal checklist applied to every sub-ticket PR-into-epic merge until
FEAT-002's canonical protocol lands:

1. Diff stays inside the ticket's declared file path hints.
2. All `## Constraints` are met (no forbidden edits, no scope creep).
3. Every acceptance criterion has a corresponding evidence line in the
   sub-agent's report or in the post-execution verification run.
4. The ticket's `## Verification` block runs green when executed on the
   epic branch.
5. Closure-CHORE edits to shared files (`SKILL.md`, `templates.md`,
   `outcome-schema.md`) do not silently revert another ticket's earlier
   edit to the same file (merge-order sanity).

### Escalation taxonomy (interim names; FEAT-002 canonicalizes)

- **`REASSIGN_NEW_SUB_AGENT`** — scope breach, repeated constraint
  violations, conceptually-wrong implementation.
- **`CORRECTIVE_SUB_TICKET`** — localized miss (missing test, minor
  constraint-adjacent issue); decompose into one TASK using the template
  landed by FEAT-001 (or, if FEAT-001 has not yet landed, use the
  Feature/Refactor template with `type: task`, `agent_created: true`,
  and a complexity score).
- **`ESCALATE_TO_PR_AGENT_WITH_FLAG`** — architectural / design
  ambiguity not resolvable from the ticket text; mark `blocked` and
  note it for the downstream PR review stage (which is itself a
  deferred ticket, not part of this epic).
- **`RESTART_FROM_SCRATCH`** — systemic risk / sweeping unscoped
  change; revert the sub-agent's branch and restart.

**Retry budget.** `MAX_FIX_CYCLES = 2` (extrapolation, see plan §4).
After two failed correction cycles, choose `REASSIGN_NEW_SUB_AGENT` or
`RESTART_FROM_SCRATCH`.

## Post-FEAT-002 switch

Once FEAT-002 has merged onto `epic/9cdd/workflow-improvements-apr26`:

- Use `.claude/skills/ticket-workflow/references/orchestrator-review-protocol.md`
  as the canonical 12-gate review checklist.
- Use `.prompts/orchestration/_template.md` as the source template for
  any future per-epic brief in other epics.
- Continue to use this per-epic file as EPIC-9cdd's authoritative
  brief; do not rewrite it from the new `_template.md` mid-epic.

## Closure

CHORE-001 runs last on this branch. On success:

1. Every ticket (including CHORE-001) is `status: done`.
2. The epic folder has moved to `.tickets/_archive/EPIC-9cdd_workflow-improvements-apr26/`.
3. This orchestration-prompt file is deleted by CHORE-001 (glob
   `epic-9cdd_*.md`; `_template.md` is intentionally underscore-prefixed
   to avoid the glob).
4. A PR from `epic/9cdd/workflow-improvements-apr26` to `main` is
   opened; the orchestrator (or the operator) cleans up only
   `.claude/worktrees/epic-9cdd/*` after merge.

## References

- Plan: `docs/implementation_plan.md`
- Adversarial review: `docs/adversarial_review_apr_2026.md`,
  `docs/adversarial_review_of_proposal_apr_2026.md`
- Research: `docs/research/r1-…r5-….md`
- Skill: `.claude/skills/ticket-workflow/SKILL.md`,
  `.claude/skills/ticket-workflow/references/templates.md`
