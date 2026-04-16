---
id: REFAC-001
title: "Complexity-scored two-layer decomposition policy"
type: refactor
status: done
priority: high
created: 2026-04-16
updated: 2026-04-16
parent: EPIC-9cdd
dependencies: []
tags: [ticket-workflow, decomposition, calibration]
agent_created: false
complexity: 5
---

# Complexity-scored two-layer decomposition policy

## Motivation

The current Execution Protocol Step 3 rule — *"If 4+ files or 5+ acceptance
criteria, decompose"* — is a crude heuristic that (a) collapses frontier-model
and Haiku-class behavior, (b) gives no calibration feedback for future
sizing, and (c) treats every "file" equally regardless of risk, cross-cutting
impact, or external-integration surface. Every later ticket in this epic
(FEAT-002 orchestrator review, FEAT-003 outcome retrieval, FEAT-004 worktree
runbook) sits near the 4-file line under the current rule and would force
ad-hoc decomposition decisions without calibrated guidance.

This refactor replaces the rule with an 8-factor weighted rubric, a
two-tier decomposition policy stated side-by-side, and a minimum-viable
calibration loop that records realized tool-round counts alongside the
predicted `complexity` score.

Source: `docs/implementation_plan.md` §2 "[REFAC] Complexity-scored two-layer
decomposition policy" (sequenced item #2); R3 §"Updated decomposition
guidance" and §"Complexity scoring approaches"; resolves adversarial review
§3 "Proposal C collapses Tier A onto Tier B", §3 "no calibration mechanism",
and §2 "grep-for-text" (structural factor-weight arithmetic check).

## Scope

Files touched:

- `.claude/skills/ticket-workflow/references/complexity-scoring.md` (create)
  — 8-factor rubric with per-factor `###` heading, band guidance (1–3 / 4–6
  / 7–8 / 9–10), and one end-to-end worked example.
- `.claude/skills/ticket-workflow/SKILL.md` (modify)
  — rewrite Step 3; append a sub-step to Step 6.
- `.claude/skills/ticket-workflow/references/templates.md` (modify)
  — update the `complexity:` comment in the five pre-existing templates
  (Feature, Bug, Chore, Refactor, Epic).

## Target structure

### New reference: `references/complexity-scoring.md`

Create the file with the 8-factor weighted rubric **verbatim from R3
§"Complexity scoring approaches"**. The eight factors and weights (must sum
to exactly 100%):

| Factor                       | Weight |
|------------------------------|--------|
| Files affected               | 20%    |
| Dependency count             | 15%    |
| Testing complexity           | 15%    |
| Risk level                   | 15%    |
| New vs modify                | 10%    |
| Cross-cutting concerns       | 10%    |
| External API integration     |  5%    |
| Database changes             | 10%    |

Each factor gets exactly one `###` heading, four-band sub-score guidance
(1–3 / 4–6 / 7–8 / 9–10), and the file ends with one worked example that
maps sub-scores to a weighted total. Close the file with one paragraph
pointing back to `SKILL.md` Step 3 so agents see where the score is
consumed.

### SKILL.md Step 3 rewrite

Replace the current body of "Step 3: Assess complexity" with exactly two
paragraphs:

> **Paragraph 1.** Populate `complexity` (1–10) in the ticket's frontmatter
> using the rubric in `references/complexity-scoring.md`. Required whenever
> the ticket has a parent set or affects more than one file.
>
> **Paragraph 2.** Decompose (proceed to Step 4) if **either** trigger fires:
> **(i)** the model-tier file threshold is exceeded. *Tier B (frontier,
> current default): decompose at complexity ≥ 8, ≥ 5 real files, or any
> high-risk cross-cutting / data / integration change. Tier A (Haiku-class):
> decompose at complexity ≥ 6, ≥ 3 real files.* **(ii)** execution stalls
> past ~25 meaningful tool rounds without convergence — abort and decompose.

Both tiers must appear side-by-side in the SKILL.md prose — not one
subsumed into the other (this is the adversarial review's specific
§3 flag).

### SKILL.md Step 6 addendum

Append one sub-step (or trailing sentence) to "Step 6: Verify":

> Record the **actual tool-round count** in the ticket's verification log
> alongside the `complexity` populated at Step 3. Future archive retrieval
> surfaces drift between predicted complexity and realized cost.

This is the minimum-viable calibration loop required by the adversarial
review.

### templates.md comment harmonization

For each of the five existing templates (Feature, Bug, Chore, Refactor,
Epic), change the `complexity:` comment from:

```
complexity:              # 1-10, optional — helps agents decide whether to decompose
```

to:

```
complexity:              # 1-10; see references/complexity-scoring.md. Required when parent set or files affected > 1
```

Do **not** touch the `TASK` template — its `complexity` comment already
carries the stricter `required for TASK` annotation from FEAT-001.

## Acceptance criteria

- [ ] Old rule deleted: `grep -c '4+ files or 5+ acceptance criteria'
      .claude/skills/ticket-workflow/SKILL.md` returns `0`.
- [ ] Rubric file exists with exactly eight weighted factors:
      `[ -f .claude/skills/ticket-workflow/references/complexity-scoring.md ]`
      **and** `grep -cE '\((20|15|10|5)%\)'
      .claude/skills/ticket-workflow/references/complexity-scoring.md`
      returns exactly `8`.
- [ ] Weights sum to 100 (structural arithmetic check):
      `python3 -c "import re; w=[int(m) for m in
      re.findall(r'\((\d+)%\)',
      open('.claude/skills/ticket-workflow/references/complexity-scoring.md'
      ).read())][:8]; assert sum(w)==100, w"` exits 0.
- [ ] Both tiers present in SKILL.md:
      `grep -cE '(complexity ≥ ?8|≥ ?5 real files)'
      .claude/skills/ticket-workflow/SKILL.md` returns ≥ 1 **and**
      `grep -cE '(complexity ≥ ?6|≥ ?3 real files)'
      .claude/skills/ticket-workflow/SKILL.md` returns ≥ 1.
- [ ] Stall trigger and calibration loop both present:
      `grep -c '25 meaningful' .claude/skills/ticket-workflow/SKILL.md`
      returns ≥ 1 **and** `grep -c 'actual tool-round count'
      .claude/skills/ticket-workflow/SKILL.md` returns ≥ 1.

## Constraints

- Pure refactor of the decomposition policy — **no** change to Step 1, 2, 4,
  5, or 7 of the Execution Protocol.
- Do NOT rename or renumber any existing SKILL.md section.
- Do NOT introduce a dependency on any external tool (the rubric is pure
  prose; the calibration loop is a single log line).
- Do NOT add automated scoring; the rubric is applied by the agent at
  ticket-creation or Step 3 assessment time.
- Do NOT change the TASK template's `complexity` comment (owned by
  FEAT-001).

## Verification

```bash
SKILL=.claude/skills/ticket-workflow/SKILL.md
RUBRIC=.claude/skills/ticket-workflow/references/complexity-scoring.md
TEMPLATES=.claude/skills/ticket-workflow/references/templates.md

# AC-1: old rule gone
[ "$(grep -c '4+ files or 5+ acceptance criteria' "$SKILL")" = "0" ]

# AC-2: rubric file + 8 weighted factors
[ -f "$RUBRIC" ]
[ "$(grep -cE '\((20|15|10|5)%\)' "$RUBRIC")" = "8" ]

# AC-3: weights sum to 100
python3 -c "import re; w=[int(m) for m in re.findall(r'\((\d+)%\)', open('$RUBRIC').read())][:8]; assert sum(w)==100, w"

# AC-4: both tiers present
grep -qE '(complexity ≥ ?8|≥ ?5 real files)' "$SKILL"
grep -qE '(complexity ≥ ?6|≥ ?3 real files)' "$SKILL"

# AC-5: stall trigger + calibration loop
grep -q '25 meaningful' "$SKILL"
grep -q 'actual tool-round count' "$SKILL"

# Sanity: templates.md comment harmonized for pre-existing templates only
#   (TASK template keeps its stricter comment — not included in this count).
[ "$(grep -c 'Required when parent set or files affected > 1' "$TEMPLATES")" -ge 5 ]
```

## Notes

- R3's inflection-point data (SWE-PolyBench ~3 files for Haiku-class, larger
  for frontier) is the anchor for the tier thresholds. Treat them as
  defaults; `docs/implementation_plan.md` §4 names the observable signals
  that would justify future adjustment.
- The "~25 meaningful tool rounds" figure is heuristic. Step 6's calibration
  loop is what makes future tuning empirical rather than speculative.
- Weights are deliberately not tunable per-agent — keep the math simple so
  the rubric stays usable during Step 3 without running a script.
