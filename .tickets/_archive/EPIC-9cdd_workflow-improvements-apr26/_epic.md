---
id: EPIC-9cdd
title: "Ticket workflow improvements (Apr 2026 synthesis)"
type: epic
status: done
priority: high
branch: epic/9cdd/workflow-improvements-apr26
created: 2026-04-16
updated: 2026-04-16
tags: [meta, ticket-workflow, tooling]
agent_created: false
complexity: 8
---

# Ticket workflow improvements (Apr 2026 synthesis)

## Context

This epic ships the set of ticket-workflow improvements synthesized in
`docs/implementation_plan.md` (Apr 16, 2026), which in turn consolidates:

- The original audit `docs/adversarial_review_apr_2026.md`.
- Research artifacts `docs/research/r1-…r5-….md` (PR review agent protocol,
  orchestrator review protocol, decomposition calibration, archive retrieval,
  worktree failure recovery).
- The adversarial review in `.prompts/adversarial/` and the resulting
  synthesis in `.prompts/synthesis/` + `docs/adversarial_review_of_proposal_apr_2026.md`.

Scope is deliberately bounded to the ticket-workflow skill
(`.claude/skills/ticket-workflow/`), its references, related runbooks, and
the orchestration prompt convention under `.prompts/orchestration/`.

What must NOT change in this epic:

- `AGENTS.md` structure (out of scope per plan §3).
- Canonical SKILL.md location / symlink conventions (dropped per plan §3).
- PR review agent protocol content (deferred per plan §2 — resurfaces as its
  own ticket after FEAT-002 lands).
- Semantic retrieval lane / persistent vector indexes (deferred per plan §2 —
  gated by the ship trigger documented in `references/outcome-schema.md`).

## Sub-tickets

| ID        | Title                                                          | Status |
|-----------|----------------------------------------------------------------|--------|
| FEAT-001  | TASK template in references/templates.md                       | to-do  |
| REFAC-001 | Complexity-scored two-layer decomposition policy               | to-do  |
| FEAT-002  | Orchestrator review-and-merge protocol                         | to-do  |
| REFAC-002 | Remove docs/INDEX.md from closure (SKILL.md + templates.md)    | to-do  |
| REFAC-003 | dependencies ID resolution scope                               | to-do  |
| REFAC-004 | Idempotency guards on closure steps (reference by name)        | to-do  |
| FEAT-003  | ## Outcome section + Lane A archive retrieval (D1)             | to-do  |
| FEAT-004  | Worktree recovery runbook content (E1)                         | to-do  |
| REFAC-005 | SKILL.md callouts + worktree serialization guard (E2)          | to-do  |
| CHORE-001 | Epic closure — archive, delete orchestration prompt, cleanup   | to-do  |

<!-- Update Status as work progresses. -->

## Merge order

<!-- Sequenced block first: authored after each other so later items cite the
     rule, template, and protocol the earlier items land. Parallel lane may
     merge in any order; E2 must come after E1. CHORE-001 is always last. -->

1. FEAT-001 — TASK template (no deps; unblocks decomposition contract for every later item).
2. REFAC-001 — Complexity-scored decomposition policy (parallel-safe with FEAT-001; must land before any parallel-lane ticket so their sizing is authored against the new rule).
3. FEAT-002 — Orchestrator review-and-merge protocol (depends on FEAT-001 + REFAC-001; consumes the TASK template and the complexity-scoring reference).
4. REFAC-002 — Remove docs/INDEX.md from closure (parallel; no dep).
5. REFAC-003 — Dependency-ID scope (parallel; soft-depends on FEAT-001 so the new TASK template inherits the annotation in one shot).
6. REFAC-004 — Closure idempotency guards (parallel; no dep, references steps by name).
7. FEAT-003 — Outcome schema + Lane A retrieval (parallel; soft-depends on REFAC-001 for sizing).
8. FEAT-004 — Worktree recovery runbook content, E1 (parallel; soft-depends on REFAC-001 for sizing).
9. REFAC-005 — SKILL.md worktree callouts + serialization guard, E2 (hard-depends on FEAT-004 — cites the runbook by section).
10. CHORE-001 — Epic closure (always last: archives epic, deletes orchestration prompt, cleans worktree).

## Acceptance criteria

- [ ] All sub-tickets are `done`.
- [ ] Every success-criterion shell check from `docs/implementation_plan.md`
      §"Section 2 — Ticket-ready items" passes on the epic branch before PR.
- [ ] Epic archived and orchestration prompt deleted by CHORE-001.
- [ ] PR from `epic/9cdd/workflow-improvements-apr26` to `main` created and
      ready for review.
- [ ] No change to `AGENTS.md`, no canonical-location symlink introduced, no
      persistent vector index checked in.

## Notes

- Source plan: `docs/implementation_plan.md` (Apr 16, 2026).
- Research sources: `docs/research/r1-…r5-….md`.
- Deferred (tracked separately, not in this epic): PR review agent protocol
  (unblocked by FEAT-002); semantic retrieval Lane B (unblocked by archive-size
  or relevance-failure trigger documented in `references/outcome-schema.md`).
- Non-goals for this epic are enumerated in `docs/implementation_plan.md` §3.
