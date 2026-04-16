# Ticket Workflow: Implementation Plan
**Date:** April 16, 2026

## Section 1 — Final sequence

1. **Add the `TASK` template to `references/templates.md`.**
   - **Why first:** All downstream decomposition events — Execution Protocol Step 4, the orchestrator's `CORRECTIVE_SUB_TICKET` escalation, and (when eventually built) the PR-review-agent ticket-materialization loop — emit `TASK` files. Without the template each decomposer improvises, voiding the "structured, consistent work units" contract that every later improvement assumes.
   - **Blocks:** The orchestrator review protocol (item 3); the PR review agent loop (deferred); every future agent-created sub-ticket.

2. **Replace the 4-files / 5-ACs rule with a complexity-scored two-layer decomposition policy.**
   - **Why second:** Every subsequent ticket in this plan (item 3, the Outcome/retrieval lane, the worktree runbook) is near-threshold under the current rule and would force ad-hoc decomposition without calibrated guidance. Landing C second — not third or later — means items 3, D1, E1, E2 are authored against the rule the proposal itself declares correct, not the rule it declares obsolete (proposal Part 3; review §6; R3 §"Updated decomposition guidance").
   - **Blocks:** Nothing hard-blocks on C, but items 3, D1, E1, E2 should be authored after it so their own sizing decisions cite the new rule.

3. **Specify the orchestrator review-and-merge protocol (R2's 12-gate checklist + 4 escalation actions + prompt template).**
   - **Why third:** The orchestrator is the first automated quality gate; every further automation (the deferred PR review agent) inherits its merge semantics. With the `TASK` template available and complexity calibration applied, the protocol ships as one cohesive ticket rather than a scope-creeping mini-epic.
   - **Blocks:** The deferred PR review agent protocol (its rejection re-entry loop assumes the orchestrator's outcome states and `CORRECTIVE_SUB_TICKET` path); every corrective-ticket escalation.

---

## Section 2 — Ticket-ready items

Ordered by ship order: three sequenced items, then parallel-lane items, then deferred items.

### [FEAT] `TASK` template in `references/templates.md`

- **Disposition:** Keep with amendments.
- **Source:** Part 1 #2; resolves review §4 "5-backtick fencing" and review §4 "`dependencies: []` annotation" (folded in here rather than duplicated in Part 1 #3's ticket).
- **Change:**
  - Append a `## Task` section to `.claude/skills/ticket-workflow/references/templates.md`, below `## Refactor` and above `## Epic`.
  - Wrap the template in a **5-backtick fenced block** (matches every existing template in the file).
  - Required frontmatter: `id: TASK-XXX`, `title: ""`, `type: task`, `status: to-do`, `parent:  # required — cannot be empty; TASKs only exist inside a parent epic`, `dependencies: []  # bare IDs resolve within current epic; cross-epic: EPIC-<hex>/TYPE-NNN`, `agent_created: true`, `complexity:  # 1-10, required for TASK; see references/complexity-scoring.md`.
  - Body: exactly five `##` subsections in order — `## Goal` (1 sentence), `## File path hints`, `## Constraints` (Do NOT rules), `## Acceptance criteria` (≤3), `## Verification` (runnable commands).
  - Omit `Context`, `Description`, `Steps to reproduce`, `Notes` — TASKs are decomposed in-session and do not stand alone.
- **Success criteria** (all shell-runnable):
  1. `grep -cn '^## Task$' .claude/skills/ticket-workflow/references/templates.md` returns `1`.
  2. `awk '/^## Task$/,/^## Epic$/' .../templates.md | grep -E '^## (Goal|File path hints|Constraints|Acceptance criteria|Verification)$' | paste -sd, -` prints exactly `## Goal,## File path hints,## Constraints,## Acceptance criteria,## Verification` (correct set and order).
  3. `awk '/^## Task$/,/^## Epic$/' .../templates.md | grep -cE 'parent: *# required|dependencies: \[\] *# bare IDs'` returns ≥2 (both annotated fields present).
  4. Template block is wrapped in 5-backtick fences: `grep -cE '^\`{5}$' .../templates.md` returns `14` after the edit (was `12` before — 5 templates × 2 fences + 2 new).
- **Sequence position:** first of three.

---

### [REFAC] Complexity-scored two-layer decomposition policy

- **Disposition:** Keep with amendments.
- **Source:** Part 2 C; resolves review §3 "Proposal C collapses Tier A onto Tier B" (both tiers stated side-by-side), review §3 "no calibration mechanism" (round-count recording added to Step 6), review §2 "grep-for-text" (structural factor-weight arithmetic check).
- **Change:**
  - Create `.claude/skills/ticket-workflow/references/complexity-scoring.md` with the 8-factor weighted rubric verbatim from R3 §"Complexity scoring approaches": files affected (20%), dependency count (15%), testing complexity (15%), risk level (15%), new vs modify (10%), cross-cutting concerns (10%), external API integration (5%), database changes (10%). Each factor gets one `###` heading, sub-score guidance for bands 1–3 / 4–6 / 7–8 / 9–10, and one worked example mapping sub-scores to a weighted total.
  - Rewrite Step 3 of `.claude/skills/ticket-workflow/SKILL.md` from "If 4+ files or 5+ acceptance criteria…" to:
    - Para 1: populate `complexity` (1–10) in frontmatter using the rubric in `references/complexity-scoring.md`.
    - Para 2: decompose if **either** trigger fires — (i) model-tier file threshold exceeded, stating both tiers side-by-side: *Tier B (frontier, current default): decompose at complexity ≥8, ≥5 real files, or any high-risk cross-cutting / data / integration change. Tier A (Haiku-class): decompose at complexity ≥6, ≥3 real files.* (ii) execution stalls past ~25 meaningful tool rounds without convergence — abort and decompose.
  - Append a sub-step to Step 6 (Verify): "Record the actual tool-round count in the ticket's verification log alongside the `complexity` populated at Step 3. Future archive retrieval surfaces drift between predicted complexity and realized cost." This is the minimum-viable calibration loop the review required.
  - In every existing template in `references/templates.md` (Feature, Bug, Chore, Refactor, Epic), change the `complexity:` comment from `# 1-10, optional` to `# 1-10; see references/complexity-scoring.md. Required when parent set or files affected > 1`.
- **Success criteria:**
  1. `grep -c '4+ files or 5+ acceptance criteria' .claude/skills/ticket-workflow/SKILL.md` returns `0`.
  2. `[ -f .claude/skills/ticket-workflow/references/complexity-scoring.md ]` exits 0 **and** `grep -cE '\((20|15|10|5)%\)' .../complexity-scoring.md` returns exactly `8`.
  3. Factor weights sum to 100: `python3 -c "import re; w=[int(m) for m in re.findall(r'\((\d+)%\)', open('.claude/skills/ticket-workflow/references/complexity-scoring.md').read())][:8]; assert sum(w)==100, w"` exits 0.
  4. Both tiers present: `grep -cE '(complexity ≥8|≥5 real files)' SKILL.md` returns ≥1 **and** `grep -cE '(complexity ≥6|≥3 real files)' SKILL.md` returns ≥1.
  5. Stall trigger + calibration loop present: `grep -c '25 meaningful' SKILL.md` returns ≥1 **and** `grep -c 'actual tool-round count' SKILL.md` returns ≥1.
- **Sequence position:** second of three.

---

### [FEAT] Orchestrator review-and-merge protocol

- **Disposition:** Keep with amendments.
- **Source:** Part 2 A; resolves review §2 "gate count / labeling scheme collide" (12-item numbered list verbatim, no A–I relabeling, including the missing item-9 "implicit behavior changes" gate), review §2 "placeholder count contradicts R2" (prompt template reproduced verbatim, no fabricated `<TICKET_PACKET>` token), review §2 "grep-for-text" (structural checks below), review §4 "Escalation pattern comparison vs decision logic" (provenance corrected), review §4 "`_template.md` excluded from closure glob" (explicit SKILL.md note), review §4 "`MAX_FIX_CYCLES=2` is extrapolation" (flagged inline as tunable).
- **Change:**
  - Create `.claude/skills/ticket-workflow/references/orchestrator-review-protocol.md` with exactly four `##` sections:
    - `## Review Gates` — reproduce R2 §"Orchestrator review checklist" items **1–12 verbatim** as one numbered list, including item 9 ("implicit behavior changes in shared or configuration surfaces"). Do not relabel as A–I.
    - `## Escalation Actions` — the four actions from R2 §"Escalation decision logic" (not §"Escalation pattern comparison" — correcting the proposal's provenance): `REASSIGN_NEW_SUB_AGENT`, `CORRECTIVE_SUB_TICKET`, `ESCALATE_TO_PR_AGENT_WITH_FLAG`, `RESTART_FROM_SCRATCH`, each with the failure-type → action rule reproduced from R2.
    - `## Retry Budget` — `MAX_FIX_CYCLES = 2` as default; one sentence noting "extrapolated from R2 which leaves it as a placeholder; see `docs/implementation_plan.md` §4 for the observable signal that would justify change".
    - `## Outcome States` — the six terminal states: `MERGED`, `NEEDS_FIX`, `REASSIGNED`, `ESCALATED`, `RESTARTED`, `BLOCKED_DEPENDENCY`.
  - Create `.prompts/orchestration/_template.md` by reproducing `docs/research/r2-orchestrator-review-protocol.md` lines 173–296 **verbatim** (the full `SYSTEM / ROLE … OUTPUT FORMAT` prompt). This preserves the complete placeholder set R2 provides (≥22 distinct tokens including `<EPIC_NAME>`, `<EPIC_BRANCH>`, `<EPIC_GOAL>`, `<EPIC_LEVEL_CONSTRAINTS>`, `<TICKET_DEPENDENCIES_AND_ORDER>`, `<TICKET_ID>`, `<TICKET_TITLE>`, `<WORKTREE_PATH>`, `<TICKET_BRANCH>`, `<SCOPE_SUMMARY>`, `<ALLOWLIST_PATHS>`, `<ACCEPTANCE_CRITERIA_LIST>`, `<CONSTRAINTS_LIST>`, `<VERIFICATION_COMMANDS>`, `<VERIFICATION_LOG>`, `<SANITY_COMMANDS>`, `<SIZE_THRESHOLDS>`, `<EXPLICIT_ALLOWED_EXCEPTIONS>`, `<MAX_FIX_CYCLES>`, `<MERGE_METHOD>`, `<POST_MERGE_CHECKS>`, `<WORKTREE_CLEANUP_STEPS>`). Prepend a 3-line HTML comment naming the file's purpose, the `<MAX_FIX_CYCLES>` default (2), and a pointer to `.claude/skills/ticket-workflow/references/orchestrator-review-protocol.md`.
  - In `.claude/skills/ticket-workflow/SKILL.md`, insert a new section `## Orchestrator Review Protocol` immediately after "Epic Branch Workflow". Content: one paragraph pointing to both new files; one bullet listing the six outcome states as `MERGED / NEEDS_FIX / REASSIGNED / ESCALATED / RESTARTED / BLOCKED_DEPENDENCY`; one sentence: "The file `.prompts/orchestration/_template.md` is underscore-prefixed so it is **not** matched by the closure ticket's `epic-<hex>_*.md` glob; do not change that glob."
- **Success criteria:**
  1. Protocol file structure: `[ -f .claude/skills/ticket-workflow/references/orchestrator-review-protocol.md ]` exits 0 **and** `grep -cE '^## (Review Gates|Escalation Actions|Retry Budget|Outcome States)$' .../orchestrator-review-protocol.md` returns exactly `4`.
  2. At least 12 numbered gates present: `awk '/^## Review Gates$/,/^## Escalation Actions$/' .../orchestrator-review-protocol.md | grep -cE '^[0-9]+\. \*\*' | awk '$1>=12{ok=1} END{exit !ok}'` exits 0.
  3. Four escalation action names present: `grep -cE 'REASSIGN_NEW_SUB_AGENT|CORRECTIVE_SUB_TICKET|ESCALATE_TO_PR_AGENT_WITH_FLAG|RESTART_FROM_SCRATCH' .../orchestrator-review-protocol.md | awk '$1>=4{ok=1} END{exit !ok}'` exits 0.
  4. Prompt template exists with ≥6 sample placeholders (the full R2 set inherits verbatim): `[ -f .prompts/orchestration/_template.md ]` **and** `grep -cE '<(EPIC_NAME|EPIC_BRANCH|SANITY_COMMANDS|SIZE_THRESHOLDS|MAX_FIX_CYCLES|MERGE_METHOD)>' .prompts/orchestration/_template.md | awk '$1>=6{ok=1} END{exit !ok}'` exits 0.
  5. SKILL.md integration: `grep -cn '^## Orchestrator Review Protocol$' SKILL.md` returns `1`; `grep -cn 'MERGED / NEEDS_FIX / REASSIGNED / ESCALATED / RESTARTED / BLOCKED_DEPENDENCY' SKILL.md` returns `1`; `grep -c '_template.md' SKILL.md` returns ≥1 (glob-exclusion note present).
- **Sequence position:** third of three.

---

### [REFAC] Remove `docs/INDEX.md` from closure (SKILL.md + templates.md)

- **Disposition:** Keep with amendments.
- **Source:** Part 1 #1; resolves review §2 "Proposal #1 leaves INDEX.md references live in templates.md" by extending the change to both files; resolves review §2 "step-numbering collision with #4" via the fix applied in the idempotency-guards ticket (steps referenced by name, not number).
- **Change:**
  - In `.claude/skills/ticket-workflow/SKILL.md`, delete the current step 4 ("Update `docs/INDEX.md`") from the "Epic Closure Ticket" numbered list and renumber remaining items so the final list has exactly five top-level entries, covering: mark done → archive → delete prompt → clean worktree → commit.
  - In `.claude/skills/ticket-workflow/references/templates.md`, remove the two `INDEX.md` mentions in the Epic template's `## Merge order` block (currently the HTML-comment on line 255 and the final bullet on line 261). Rewrite the comment and the final bullet to describe the closure CHORE as "archives epic, deletes orchestration prompt" with no reference to any external index file.
- **Success criteria:**
  1. `grep -c 'INDEX.md' .claude/skills/ticket-workflow/SKILL.md` returns `0`.
  2. `grep -c 'INDEX.md' .claude/skills/ticket-workflow/references/templates.md` returns `0`.
  3. Closure list has exactly five top-level items: `awk '/^## Epic Closure Ticket$/,/^## /' SKILL.md | grep -cE '^[0-9]+\. \*\*' | awk '$1==5{ok=1} END{exit !ok}'` exits 0.
- **Sequence position:** parallel lane.

---

### [REFAC] `dependencies` ID resolution scope

- **Disposition:** Keep with amendments.
- **Source:** Part 1 #3; resolves review §4 "off-by-one — Epic template has no `dependencies:` field" by deriving the expected count from the file itself.
- **Change:**
  - Under the "Naming" section of `.claude/skills/ticket-workflow/SKILL.md`, add a subsection `### Dependency ID resolution` (2–3 sentences): "Bare IDs in a ticket's `dependencies` field resolve within the current epic. Cross-epic dependencies must use the full form `EPIC-<hex>/TYPE-NNN`. Agents resolving a bare ID that has no in-epic match must fail the dependency check, not search globally."
  - In `.claude/skills/ticket-workflow/references/templates.md`, for every template that has a `dependencies: []` field (Feature, Bug, Chore, Refactor, and the new TASK once Part 1 #2 lands — **not** Epic), change the comment to `dependencies: []         # bare IDs resolve within current epic; cross-epic: EPIC-<hex>/TYPE-NNN`.
- **Success criteria:**
  1. `grep -c 'bare IDs resolve within current epic' .claude/skills/ticket-workflow/SKILL.md` returns ≥1.
  2. Annotation count matches `dependencies:` field count (self-adjusting): `expected=$(grep -c '^dependencies: \[\]' .claude/skills/ticket-workflow/references/templates.md); actual=$(grep -c 'bare IDs resolve within current epic; cross-epic: EPIC-<hex>/TYPE-NNN' .claude/skills/ticket-workflow/references/templates.md); [ "$expected" = "$actual" ]`.
  3. Epic template remains unaffected: `awk '/^## Epic$/,/^## /' templates.md | grep -c 'dependencies:'` returns `0`.
- **Sequence position:** parallel lane.

---

### [REFAC] Idempotency guards on closure steps (reference by name, not number)

- **Disposition:** Keep with amendments.
- **Source:** Part 1 #4; resolves review §2 "step-numbering collision with #1" by referring to steps by name rather than index.
- **Change:**
  - Rewrite each destructive step in the "Epic Closure Ticket" section of `.claude/skills/ticket-workflow/SKILL.md` to embed its existence check, identifying the step by its purpose rather than by its numbered position:
    - *Archive step*: `[ -d .tickets/_archive/EPIC-<hex>_<slug> ] || git mv .tickets/EPIC-<hex>_<slug> .tickets/_archive/EPIC-<hex>_<slug>`
    - *Prompt-deletion step*: `[ -f .prompts/orchestration/epic-<hex>_*.md ] && git rm .prompts/orchestration/epic-<hex>_*.md || true`
    - *Worktree-cleanup step*: `[ -d .claude/worktrees/epic-<hex> ] && rm -rf .claude/worktrees/epic-<hex> || true`
  - Add one explicit sentence at the top of the section: "Every step below must be safe to re-run; check existence before acting."
- **Success criteria:**
  1. `grep -c '\[ -d .tickets/_archive/EPIC-<hex>_<slug> \]' .claude/skills/ticket-workflow/SKILL.md` returns ≥1.
  2. `grep -c '\[ -f .prompts/orchestration/epic-<hex>_\*.md \]' .claude/skills/ticket-workflow/SKILL.md` returns ≥1.
  3. `grep -c '\[ -d .claude/worktrees/epic-<hex> \]' .claude/skills/ticket-workflow/SKILL.md` returns ≥1.
  4. `grep -c 'safe to re-run' .claude/skills/ticket-workflow/SKILL.md` returns ≥1.
- **Sequence position:** parallel lane.

---

### [FEAT] `## Outcome` section + Lane A archive retrieval (D1)

- **Disposition:** Split — this ticket ships Lane A; Lane B is a separate deferred ticket below.
- **Source:** Part 2 D Lane A only; resolves review §3 "Step 7 sub-step ambiguity" (explicit commit policy), review §3 "scale horizon not named" (~2,000-chunk trigger called out inline), review §2 "grep-for-text" (structural subsection-header count + end-to-end fixture test), review §4 "`--help` passes with a stub" (fixture-based end-to-end AC).
- **Change:**
  - Create `.claude/skills/ticket-workflow/references/outcome-schema.md`:
    - Reproduce the `## Outcome` schema verbatim from R4 §"Structured outcome section proposal": seven bolded subsections in this order — `**Summary:**`, `**Key decisions:**`, `**Constraints & invariants discovered (keep):**`, `**Implementation notes (high signal only):**`, `**Verification:**`, `**Risk / regression surface:**`, `**Retrieval tags:**`. Target length **120–200 words / ≤8 short bullets**.
    - Include the filled "request-scoped cache" example from R4.
    - Add one closing paragraph: *"When `.tickets/_archive/` exceeds ~2,000 `## Outcome` chunks (≈200 tickets at ~10 bullets each), Lane A precision will degrade; at that point ship Lane B per `docs/implementation_plan.md` §4 trigger. Until then, Lane A alone is sufficient. See `docs/research/r4-archive-knowledge-retrieval.md` §'Upgrade path as the archive grows'."*
  - In `.claude/skills/ticket-workflow/SKILL.md`, insert a new first sub-step in Step 7 before "Set `status: done`": "Append an `## Outcome` section to the ticket using the schema in `references/outcome-schema.md`." Immediately after that sub-step, add a commit-policy clarification: *"The Outcome append is staged in the same commit as `status: done` (the mark-done commit). The 'separate from implementation work' rule refers to **production code changes**, not ticket metadata written at closure."*
  - In `.claude/skills/ticket-workflow/SKILL.md`, append a new top-level section `## Querying past work` (below "Epic Closure Ticket"): one paragraph directing agents to run `bash .claude/skills/ticket-workflow/scripts/archive-search.sh '<short epic pitch>'` before creating a new epic touching unfamiliar subsystems, and to inject top matches' `## Outcome` blocks into the new epic's Context section.
  - Create `.claude/skills/ticket-workflow/scripts/archive-search.sh` implementing **Lane A only**:
    - Shebang `#!/usr/bin/env bash`, executable bit set, `set -euo pipefail`.
    - `--help` prints usage: query syntax, `--type`, `--complexity`, `--tags` flags, and one line "Lane B (--semantic) is not yet available; see `.claude/skills/ticket-workflow/references/outcome-schema.md` for the ship trigger."
    - Default: `rg -l --type md` over `.tickets/_archive/` to locate files whose YAML frontmatter or `## Outcome` block matches the query. Flag filters apply as frontmatter post-filters. Output = matching paths + their `## Outcome` section snippets; never the full ticket body.
    - Empty archive: print "no matches" to stdout and exit 0 — smoke tests must not false-fail.
- **Success criteria:**
  1. Schema shape: `[ -f .claude/skills/ticket-workflow/references/outcome-schema.md ]` **and** `grep -cE '^\*\*(Summary|Key decisions|Constraints & invariants discovered \(keep\)|Implementation notes \(high signal only\)|Verification|Risk / regression surface|Retrieval tags):\*\*' .../outcome-schema.md` returns exactly `7`.
  2. Scale-horizon sentence present: `grep -cE '2,?000 \`?## Outcome\`? chunks' .../outcome-schema.md` returns ≥1.
  3. Script `--help` works: `bash .claude/skills/ticket-workflow/scripts/archive-search.sh --help` exits 0 and output contains both `--semantic` and the "not yet available" message.
  4. End-to-end fixture test: the following one-liner exits 0 and prints the fixture path —
     ```bash
     TMP=$(mktemp -d); mkdir -p "$TMP/.tickets/_archive/EPIC-test_x"; printf -- '---\ntype: feature\ntags: [cache]\n---\n## Outcome\n**Summary:** fixture.\n**Retrieval tags:** cache\n' > "$TMP/.tickets/_archive/EPIC-test_x/FEAT-001_y.md"; (cd "$TMP" && bash "$OLDPWD/.claude/skills/ticket-workflow/scripts/archive-search.sh" cache) | grep -q 'FEAT-001_y.md'
     ```
  5. SKILL.md integration: `grep -cn '^## Querying past work$' SKILL.md` returns `1`; `grep -c 'references/outcome-schema.md' SKILL.md` returns ≥1; `grep -c 'production code changes' SKILL.md` returns ≥1 (commit-policy clarification present).
- **Sequence position:** parallel lane, after item 2 (complexity policy) lands so its sizing is calibrated.

---

### [FEAT] Worktree recovery runbook content (E1)

- **Disposition:** Split — E1 authors the runbook; E2 (below) wires it into SKILL.md.
- **Source:** Part 2 E content; resolves review §3 "runbook will not fit in a single ticket" (split from SKILL.md integration), review §3 "10-row table description undercounts R5's 12" (specifies 12 rows), review §3 "serialization rule unenforceable" (pairs the rule with a retry+jitter shell snippet in the Prevention section).
- **Change:**
  - Create `docs/runbooks/worktree-recovery.md` with three top-level sections:
    - `## Diagnostic commands` — the **12-row** table verbatim from R5 §"Quick-reference diagnostic table": `git worktree list --porcelain`, `git worktree list --verbose`, `git rev-parse --show-toplevel`, `git branch --show-current`, `git status --porcelain=v2 --branch`, `git log --oneline -5 --decorate`, `git diff --name-status`, `git diff --cached --name-status`, `git stash list`, `git rev-parse --git-dir`, `git rev-parse --git-path index` / `index.lock`, `git worktree prune --dry-run --verbose`. Each row carries R5's "what it reveals / interpret / safety" columns.
    - `## Recovery procedures` with four `###` subsections named exactly: `Sub-agent killed mid-commit`, `Closure ticket partial execution`, `Worktree in inconsistent state`, `Epic branch significantly diverged from main`. Each reproduces R5's diagnose → stabilize → repair → verify sequence, Goal A / Goal B split for closure, and the `git restore --staged --worktree :/` / `git worktree repair` / `git worktree prune` commands as appropriate.
    - `## Prevention conventions` with R5's six rules, including this shell snippet for the serialization rule so it becomes a mechanism rather than a policy:
      ```bash
      # Serialize git worktree add; retry on .git/config.lock contention with jitter.
      for attempt in 1 2 3 4 5; do
        if git worktree add "$path" -b "$branch" "$base" 2>/tmp/werr; then
          break
        fi
        grep -q 'config.lock' /tmp/werr || { cat /tmp/werr >&2; exit 1; }
        sleep "$(awk -v s="$attempt" 'BEGIN{srand(); print s*0.2 + rand()*0.3}')"
      done
      ```
- **Success criteria:**
  1. `[ -f docs/runbooks/worktree-recovery.md ]` exits 0.
  2. Diagnostic table has ≥12 command rows: `awk '/^## Diagnostic commands$/,/^## Recovery procedures$/' docs/runbooks/worktree-recovery.md | grep -cE '^\| *\`git ' | awk '$1>=12{ok=1} END{exit !ok}'` exits 0.
  3. Four recovery procedure headings present in the correct form: `grep -cE '^### (Sub-agent killed mid-commit|Closure ticket partial execution|Worktree in inconsistent state|Epic branch significantly diverged from main)$' docs/runbooks/worktree-recovery.md` returns exactly `4`.
  4. Retry+jitter mechanism present: `grep -c 'config.lock' docs/runbooks/worktree-recovery.md` returns ≥1 **and** `grep -cE 'jitter|rand\(\)' docs/runbooks/worktree-recovery.md` returns ≥1.
- **Sequence position:** parallel lane, after item 2.

---

### [REFAC] SKILL.md callouts + worktree serialization guard (E2)

- **Disposition:** Keep with amendments (split out of Part 2 E).
- **Source:** Part 2 E SKILL.md integration; depends on E1.
- **Change:**
  - In `.claude/skills/ticket-workflow/SKILL.md`:
    - At the top of the "Worktree Rules" section: **Recovery:** If any `git worktree` step fails, stop and follow `docs/runbooks/worktree-recovery.md` § Diagnostic commands before retrying.
    - At the top of the "Epic Closure Ticket" section: **Recovery:** If closure fails mid-run, follow `docs/runbooks/worktree-recovery.md` § Recovery procedures > Closure ticket partial execution (Goal A is the safe default).
    - In "Worktree Rules > Key constraints", add one bullet: **Serialize** `git worktree add` calls per repo. On `.git/config.lock` contention, retry with jitter — see `docs/runbooks/worktree-recovery.md` § Prevention conventions for the exact snippet.
- **Success criteria:**
  1. Exactly two references to the runbook: `grep -c 'docs/runbooks/worktree-recovery.md' .claude/skills/ticket-workflow/SKILL.md` returns `2`.
  2. Serialization rule present: `grep -cE '(Serialize|serialize) +\`git worktree add\`' .claude/skills/ticket-workflow/SKILL.md` returns ≥1.
  3. Lock mechanism pointer present: `grep -c '.git/config.lock' .claude/skills/ticket-workflow/SKILL.md` returns ≥1.
- **Sequence position:** parallel lane, depends on E1.

---

### [FEAT] PR review agent protocol and rejection re-entry loop (deferred)

- **Disposition:** Defer until item 3 (orchestrator review protocol) lands.
- **Source:** Part 2 B; when unblocked, resolves review §4 "grep `pr-review-agent` ≥1 would pass today" (tightens to a section-header grep) and review §2 "grep-for-text" (validates the JSON schema by parsing it).
- **Precondition to ship:** Item 3 is merged (the six orchestrator outcome states and the `CORRECTIVE_SUB_TICKET` escalation path are referenced by this protocol's rejection re-entry flow).
- **Change (for the eventual ticket):**
  - Create `.claude/skills/ticket-workflow/references/pr-review-agent-protocol.md` with three `## Tier` headings (Tier 0 / Tier 1 / Tier 2) and the checklist items from R1 §"Recommended protocol > Checklist in priority order"; the JSON rejection payload schema in a fenced `json` block — keys `verdict`, `epic_branch`, `base_branch`, `head_sha`, `findings[].{id,severity,category,title,evidence,fix_strategy,ticket_blueprint}`, `dedupe_fingerprint`, `recheck_instructions` verbatim from R1 §"Machine-readable payload schema"; the seven-step rejection re-entry flow; the dedupe rule with `N = 3` flagged as a tunable default.
  - Edit `.claude/skills/ticket-workflow/SKILL.md`: add a new top-level section `## PR Review Agent Handoff` after "Orchestrator Review Protocol". Content: required status check `pr-review-agent` blocks merge to main; orchestrator parses the JSON payload and materializes tickets (one `TASK` per BLOCKER finding) before re-requesting review.
- **Success criteria (for the eventual ticket):**
  1. Three Tier headings: `grep -cE '^## Tier [012]' .../pr-review-agent-protocol.md` returns exactly `3`.
  2. JSON schema parseable with required keys: the following one-liner exits 0 —
     ```bash
     python3 -c "import json,re,sys; t=open('.claude/skills/ticket-workflow/references/pr-review-agent-protocol.md').read(); m=re.search(r'\`\`\`json\n(.*?)\n\`\`\`', t, re.S); d=json.loads(m.group(1)); req={'verdict','epic_branch','findings','dedupe_fingerprint','recheck_instructions'}; sys.exit(0 if req<=set(d) else 1)"
     ```
  3. SKILL.md section header present: `grep -cE '^## PR Review Agent Handoff$' SKILL.md` returns exactly `1`.
  4. Dedupe default surfaced: `grep -cE 'N = 3|N=3' .../pr-review-agent-protocol.md` returns ≥1.
- **Sequence position:** deferred until item 3 lands.

---

### [FEAT] Semantic retrieval lane (D2, deferred)

- **Disposition:** Defer (split out of Part 2 D Lane B).
- **Source:** Part 2 D Lane B; resolves review §3 "scale horizon not named" by tying ship to a concrete, observable trigger rather than making Lane B speculative infrastructure for a workspace that has no archived epics yet.
- **Precondition to ship:** Either (a) `rg -c '^## Outcome$' .tickets/_archive/ | awk -F: '{s+=$2} END{print s}'` ≥ **2,000** — the R4 breakpoint where lexical precision starts to lose to semantic; **or** (b) two logged orchestrator runs report Lane A returned ≥ 5 matches but none were judged relevant. Either trigger independently justifies ship; the trigger itself is re-litigable by a future adversarial cycle.
- **Change (for the eventual ticket):**
  - Add a `--semantic` flag to `.claude/skills/ticket-workflow/scripts/archive-search.sh` that, at query time, builds an **in-session** (not persisted) `sentence-transformers/all-MiniLM-L6-v2` embedding index over `## Outcome` blocks in `.tickets/_archive/`, runs cosine similarity against the query, and returns top-3 matches. Model is overridable via `ARCHIVE_SEARCH_MODEL` env var. Honors the no-persistent-infrastructure binding constraint: nothing is written to disk.
  - Append a paragraph to `.claude/skills/ticket-workflow/references/outcome-schema.md` documenting the `--semantic` lane, default model, and override env var.
- **Success criteria (for the eventual ticket):**
  1. End-to-end fixture: `bash archive-search.sh --semantic 'caching with invalidation'` against a fixture archive containing one cache-related Outcome block returns that ticket's path within the top-3 results.
  2. `--help` documents the env override: `bash archive-search.sh --help | grep -c 'ARCHIVE_SEARCH_MODEL'` returns ≥1.
  3. No persistent artifacts: running `--semantic` twice in a row leaves `.tickets/_archive/` unchanged under `git status --porcelain`.
- **Sequence position:** deferred until precondition.

---

## Section 3 — Explicit non-goals

- **Canonical SKILL.md location / workspace symlink convention (proposal Part 1 #5).** Dropped outright. Shipping the proposal's direction (canonical = `~/.agents/skills/ticket-workflow/`) breaks self-containment — a fresh workspace clone has a dangling symlink — and creates a genuine CHORE-update paradox: a worktree is scoped to the repo, a symlink's target outside the repo is not tracked by git, and any edit to the target is invisible to any commit the CHORE tries to make (review §3). Reversing the direction (canonical = in-repo) does not generalize, because each workspace would claim canonicality with no arbitrator. The drift risk the proposal targets is Severity: Low in the original audit (`docs/adversarial_review_apr_2026.md` §Defect #7); the cure as written is worse than the disease. Per-workspace copies remain independent; cross-workspace sync, if desired, is the user's concern outside the skill. Re-litigable if drift is observed in practice across > 2 workspaces.

- **ID assignment race mitigation (original audit `docs/adversarial_review_apr_2026.md` §Defect #5).** Deferred, not dropped. The proposal silently omitted this; review §3 flagged the omission. Rationale for continued deferral: worktree isolation already largely contains the race; the original severity is Low-Medium; no in-the-wild collision has been reported. Precondition to resurface: a single observed collision, or sustained concurrent epic-creation (> 2 per week by different agents) which elevates the race probability materially.

- **Always-on proactive archive injection into every new epic.** R4 §"Proactive injection vs on-demand retrieval" warns that long-context performance degrades when relevant content is buried mid-prompt. The plan ships opt-in retrieval only (via the `## Querying past work` section of SKILL.md). Re-litigable when Outcome blocks are consistently < 50 words each and top-3 injection demonstrably fits without context pressure.

- **Persistent vector indexes (FAISS on disk, ChromaDB persistent).** Forbidden by binding constraint #1 of this synthesis. R4's upgrade path to a "rebuildable index artifact" is still on-demand computation and remains available as a future ticket; building it now would violate the constraint. Re-litigable if cold-workspace build time for `--semantic` exceeds ~30s.

- **AGENTS.md rewrite.** Out of scope. The original audit listed `Recreate AGENTS.md` as Immediate Fix #6; none of the ten proposal items touched it, and this synthesis is scoped to what came through the proposal. Re-litigable if any ticket produced by this plan surfaces an AGENTS.md-level ambiguity.

---

## Section 4 — Open decisions left for implementation

Every value below is a **default**. Where a number is not empirically grounded, that is stated.

- **`N = 3`** — dedupe-fingerprint trigger for the PR review agent's meta-ticket escalation (same fingerprint failing 3 times forces a single invariant-adding meta-ticket rather than another symptom-level fix).
  - **Rationale:** extrapolation. R1 §"Dedupe and loop prevention" leaves `N` as a placeholder without binding. Three gives two retries for transient flakiness while still capping the loop.
  - **Observable signal to change:** if dedupe-triggered meta-tickets land > 1 per epic, `N` is too low; if the same fingerprint recycles through all three slots in > 50% of observed runs, `N` is too high.

- **`MAX_FIX_CYCLES = 2`** — orchestrator corrective-loop cap before forcing `REASSIGN_NEW_SUB_AGENT`.
  - **Rationale:** extrapolation. R2 uses `<MAX_FIX_CYCLES>` as a placeholder. Two permits "one mistake + one correction" without enabling the long drift loops R3 §"Failure mode taxonomy" identifies as the single most expensive pattern.
  - **Observable signal to change:** if `REASSIGN_NEW_SUB_AGENT` rates exceed ~20 % of ticket runs, lower the cap or widen the corrective-ticket scope check. If corrective sub-tickets routinely succeed on attempt 3 in adjacent projects, raise to 3.

- **`sentence-transformers/all-MiniLM-L6-v2`** — default semantic retrieval model when Lane B (D2) ships.
  - **Rationale:** R4's named default: small (~80 MB), CPU-fast, widely cited. User-overridable via `ARCHIVE_SEARCH_MODEL`.
  - **Observable signal to change:** if cold-workspace build time consistently exceeds ~30 s, switch to a quantized variant. If semantic matches rank below lexical matches at the same query, try a code-aware embedding (e.g., `microsoft/codebert-base`).

- **Tier B (frontier) decomposition thresholds: complexity ≥ 8 **or** ≥ 5 real files **or** any high-risk cross-cutting / data / integration change.**
  **Tier A (Haiku-class): complexity ≥ 6 **or** ≥ 3 real files.**
  - **Rationale:** R3 §"Updated decomposition guidance" anchored to SWE-PolyBench's observed ~3-file inflection. The workspace defaults to Tier B.
  - **Observable signal to change:** the calibration loop added by item 2 records realized tool-round counts; if Tier B tickets routinely exhaust the 25-round budget below complexity 8, lower the threshold. Review telemetry at ~20 closed tickets.

- **Archive Lane B (D2) ship trigger: ~2,000 `## Outcome` chunks in `.tickets/_archive/`** (≈ 200 closed tickets × ~10 bullets each), **or** two logged Lane A queries that return ≥ 5 matches with none judged relevant.
  - **Rationale:** R4 §"Upgrade path" names the 2,000 figure as the empirical point where lexical precision starts to degrade.
  - **Observable signal to change:** a single Lane A query exceeding 2 s on a warm workspace, or returning > 30 paths without useful filter narrowing, justifies shipping D2 regardless of the chunk count.

- **`complexity` frontmatter field: required for every `TASK` ticket and any FEAT / REFAC / BUG with more than one file affected; optional elsewhere.**
  - **Rationale:** R3 §"When to populate the `complexity` field" recommends populating after a quick recon pass. Making it mandatory everywhere adds friction on trivial CHOREs; making it optional on multi-file work defeats the calibration loop.
  - **Observable signal to change:** if `complexity` is frequently absent on tickets that later stall past 25 rounds, escalate to mandatory for every FEAT / REFAC / BUG.

- **`~25 meaningful tool rounds` — stall trigger for mid-execution decomposition.**
  - **Rationale:** R3's figure. "Meaningful" is defined here as tool calls that *act* on the repo (read, edit, run verification, git operations) and excludes meta-queries (listing terminals, re-reading the current ticket). This is heuristic; the calibration loop's telemetry will refine it.
  - **Observable signal to change:** if decomposition-at-stall events cluster at < 10 or > 40 rounds, the threshold is misaligned and should be re-derived from telemetry rather than retained.
