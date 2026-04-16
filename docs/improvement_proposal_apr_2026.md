# Ticket Workflow Improvement Proposal
**Date:** April 16, 2026
**Inputs:** `docs/adversarial_review_apr_2026.md`; `docs/research/r1-pr-review-agent.md`; `docs/research/r2-orchestrator-review-protocol.md`; `docs/research/r3-decomposition-calibration.md`; `docs/research/r4-archive-knowledge-retrieval.md`; `docs/research/r5-worktree-failure-recovery.md`; current `.claude/skills/ticket-workflow/SKILL.md` + `references/templates.md`; current `AGENTS.md`
**Purpose:** Rank-ordered, ticket-ready improvement proposals synthesizing the adversarial review and all five research reports.

---

## Part 1: Immediate fixes

### 1. Remove `docs/INDEX.md` step from closure ticket spec

- **Type:** REFAC
- **Priority:** high
- **Problem:** `SKILL.md` § "Epic Closure Ticket" step 4 instructs every closing epic to update `docs/INDEX.md` — a workspace-specific artifact accidentally pulled into a portable skill. In any workspace lacking that file, every epic closure ticket is partially unimplementable.
- **Change:** In `.claude/skills/ticket-workflow/SKILL.md`, delete step 4 from the "Epic Closure Ticket" section and renumber steps 5–6 to 4–5. The archive itself is the completion record; no external index is required for the general case.
- **Success criteria:**
  - `grep -n 'docs/INDEX.md' .claude/skills/ticket-workflow/SKILL.md` returns no matches.
  - The "Epic Closure Ticket" numbered list contains exactly five steps.
  - The remaining five steps still cover: mark done, archive folder, delete prompt, clean worktree, commit.

---

### 2. Add TASK template to `references/templates.md`

- **Type:** FEAT
- **Priority:** high
- **Problem:** `SKILL.md` documents `TASK` as a valid type (used during decomposition in Step 4) but `references/templates.md` has no TASK template. Every decomposing agent improvises the format, undermining the "structured, consistent work units" guarantee.
- **Change:** Append a TASK template section to `.claude/skills/ticket-workflow/references/templates.md`. Lean schema: required frontmatter `id`, `title`, `type: task`, `status`, `parent` (required, never empty), `dependencies`, `agent_created: true`, `complexity`. Body sections: `## Goal` (1 sentence), `## File path hints`, `## Constraints` (Do NOT rules), `## Acceptance criteria` (≤3), `## Verification` (runnable commands). Omit Context, Notes, Description, Steps to reproduce — TASKs are decomposed in-session and don't stand alone.
- **Success criteria:**
  - `grep -n '^## Task$' .claude/skills/ticket-workflow/references/templates.md` returns one match.
  - Template contains a fenced markdown block with `parent:` listed as a required field and an instruction comment marking it required.
  - Template body lists exactly five `##` subsections in this order: Goal, File path hints, Constraints, Acceptance criteria, Verification.

---

### 3. Specify `dependencies` ID resolution scope

- **Type:** REFAC
- **Priority:** medium
- **Problem:** Sub-ticket IDs are scoped per epic (`FEAT-001` exists in every epic). When a ticket lists `dependencies: [FEAT-002]`, the SKILL.md doesn't say whether bare IDs resolve within the current epic or globally. An agent reading the wrong ticket may proceed without knowing.
- **Change:** Add a "Dependency ID Resolution" subsection to the "Naming" section of `.claude/skills/ticket-workflow/SKILL.md`. State explicitly: bare IDs in `dependencies` resolve within the current epic; cross-epic dependencies must use the full `EPIC-<hex>/TYPE-NNN` form. Mirror this rule in every template in `references/templates.md` by changing the example from `dependencies: []` to `dependencies: []  # bare IDs resolve within current epic; cross-epic: EPIC-<hex>/TYPE-NNN`.
- **Success criteria:**
  - `grep -c 'bare IDs' .claude/skills/ticket-workflow/SKILL.md` returns ≥1.
  - `grep -c 'cross-epic: EPIC-<hex>/TYPE-NNN' .claude/skills/ticket-workflow/references/templates.md` equals the number of templates (≥5).

---

### 4. Add idempotency guards to closure ticket spec

- **Type:** REFAC
- **Priority:** medium
- **Problem:** The Epic Closure Ticket runs sequential steps (`git mv`, `git rm`, `rm -rf`). If it fails partway, re-running fails on already-completed steps, leaving the epic in a half-archived state with no clean recovery.
- **Change:** In `.claude/skills/ticket-workflow/SKILL.md` "Epic Closure Ticket" section, prefix each step with a guard pattern. Step 2 (archive): `[ -d .tickets/_archive/EPIC-<hex>_<slug> ] || git mv .tickets/EPIC-<hex>_<slug> .tickets/_archive/EPIC-<hex>_<slug>`. Step 3 (delete prompt): `[ -f .prompts/orchestration/epic-<hex>_*.md ] && git rm .prompts/orchestration/epic-<hex>_*.md || true`. Step 5 (cleanup): `[ -d .claude/worktrees/epic-<hex> ] && rm -rf .claude/worktrees/epic-<hex> || true`. Add an explicit sentence: "Each step must be safe to re-run; check before acting."
- **Success criteria:**
  - Each of the three destructive commands in the closure section is preceded by an existence check.
  - `grep -c 'safe to re-run' .claude/skills/ticket-workflow/SKILL.md` returns ≥1.

---

### 5. Document canonical SKILL.md location and symlink convention

- **Type:** CHORE
- **Priority:** medium
- **Problem:** The skill currently exists at both `~/.agents/skills/ticket-workflow/SKILL.md` and `<workspace>/.claude/skills/ticket-workflow/SKILL.md`. They drift silently when an agent edits the local copy via a CHORE ticket. There is no documented canonical source.
- **Change:** Add a "Canonical Source" section after the H1 header in `.claude/skills/ticket-workflow/SKILL.md`. State: the canonical source is `~/.agents/skills/ticket-workflow/`; workspace copies under `.claude/skills/ticket-workflow/` should be symlinks to the canonical directory; CHOREs that update the skill must edit the canonical source first, then verify all workspace symlinks resolve. Include the verification command: `readlink -f .claude/skills/ticket-workflow`.
- **Success criteria:**
  - `grep -n 'Canonical Source' .claude/skills/ticket-workflow/SKILL.md` returns one match within the first 30 lines.
  - The section contains the literal path `~/.agents/skills/ticket-workflow/` and the verification command.

---

## Part 2: Research-backed improvements

### A. Specify the orchestrator review-and-merge protocol

- **Type:** FEAT
- **Priority:** critical
- **Problem:** SKILL.md describes an orchestrator that creates epics and eventually a PR, but the most consequential action — reviewing each completed sub-ticket worktree and deciding whether to merge it to the epic branch — is entirely undefined. Without protocol, orchestrators either auto-merge everything (no first-line review) or stall waiting for human direction.
- **Change:** Create new file `.claude/skills/ticket-workflow/references/orchestrator-review-protocol.md` containing the 12 ordered review gates from R2 (A: entry completeness; B: minimal sanity verification re-run; C: diff size classification with `>400 LOC = high scrutiny`; D: changed-paths ⊆ allowlist; E: hard constraints check; F: acceptance-criteria evidence matrix; G: omission gate; H: maintainability spot-check; I: rebase against latest epic; partial-merge dependency rule; final decision record). Include the four escalation actions (`REASSIGN_NEW_SUB_AGENT`, `CORRECTIVE_SUB_TICKET`, `ESCALATE_TO_PR_AGENT_WITH_FLAG`, `RESTART_FROM_SCRATCH`) and the failure-type → action decision logic. Include a `MAX_FIX_CYCLES` cap (default 2) before forcing reassignment to prevent infinite correction loops. Also create `.prompts/orchestration/_template.md` containing the full prompt template from R2 § "Orchestrator prompt template" with `<EPIC_NAME>`, `<EPIC_BRANCH>`, `<TICKET_PACKET>`, `<SANITY_COMMANDS>`, `<SIZE_THRESHOLDS>` placeholders. Add to `.claude/skills/ticket-workflow/SKILL.md` a new section "Orchestrator Review Protocol" (after "Epic Branch Workflow") with one paragraph pointing to both files and listing the six possible decision outcomes (`MERGED / NEEDS_FIX / REASSIGNED / ESCALATED / RESTARTED / BLOCKED_DEPENDENCY`).
- **Success criteria:**
  - `.claude/skills/ticket-workflow/references/orchestrator-review-protocol.md` exists and contains exactly 12 numbered gates plus the four escalation actions.
  - `.prompts/orchestration/_template.md` exists with all six placeholder tokens present.
  - `grep -n 'MERGED / NEEDS_FIX / REASSIGNED / ESCALATED / RESTARTED / BLOCKED_DEPENDENCY' .claude/skills/ticket-workflow/SKILL.md` returns one match.
- **Research basis:** R2 § "Orchestrator review checklist" enumerates the 12 ordered gates with concrete pass/fail criteria; § "Escalation pattern comparison" defines the four actions; § "Orchestrator prompt template" provides a copy-ready model-agnostic prompt with named placeholder slots.

---

### B. Specify the PR review agent protocol and rejection re-entry loop

- **Type:** FEAT
- **Priority:** critical
- **Problem:** The fully-agentic model designates the PR review agent as the terminal quality gate, but SKILL.md says nothing about what it checks, how it signals rejection, or how rejections re-enter the ticket system. If this gate is weak, the entire pipeline's quality guarantee is weak.
- **Change:** Create new file `.claude/skills/ticket-workflow/references/pr-review-agent-protocol.md` containing: (1) the staged checklist from R1 — Tier 0 (CI green, no secrets/debug artifacts, policy compliance), Tier 1 (contract surface audit, cross-ticket consistency, test adequacy), Tier 2 (security/performance/evolvability with judge-filter); (2) the JSON rejection payload schema verbatim from R1 § "Machine-readable payload schema" (`verdict`, `epic_branch`, `head_sha`, `findings[].{id,severity,category,title,evidence,fix_strategy,ticket_blueprint}`, `dedupe_fingerprint`, `recheck_instructions`); (3) the seven-step rejection re-entry flow (status check `pr-review-agent=failure` → orchestrator parses payload → materializes one ticket per BLOCKER finding → executes → updates PR head → triggers re-review); (4) dedupe rule: if same `dedupe_fingerprint` fails N=3 times, escalate by generating a single meta-ticket adding a missing invariant test instead of re-fixing symptoms. In `.claude/skills/ticket-workflow/SKILL.md`, add a "PR Review Agent Handoff" section (after "Epic Closure Ticket") pointing to the protocol file and stating: merge to main is blocked by required status check `pr-review-agent`; orchestrator must materialize tickets from the JSON payload before re-requesting review.
- **Success criteria:**
  - `.claude/skills/ticket-workflow/references/pr-review-agent-protocol.md` exists with three labeled tiers and the JSON schema as a fenced code block.
  - The JSON schema contains the keys `verdict`, `findings`, `dedupe_fingerprint`, `recheck_instructions`.
  - `grep -n 'pr-review-agent' .claude/skills/ticket-workflow/SKILL.md` returns ≥1 match.
- **Research basis:** R1 § "Recommended protocol" provides the three-tier checklist, the exact JSON payload schema (with severity levels BLOCKER/MAJOR/MINOR/INFO and `fix_strategy: NEW_TICKET` field), and the seven-step rejection re-entry flow including dedupe fingerprint loop-prevention.
- **Open questions:** (1) What N value to use for the dedupe meta-ticket trigger — R1 says "N times" without binding; pick N=3 as default but document it as tunable. (2) Whether to allow PR review agent to push commits directly for trivial fixes (R1 leans no); recommendation: defer this and start ticket-only.

---

### C. Replace decomposition heuristic with complexity-scored two-layer policy

- **Type:** REFAC
- **Priority:** high
- **Problem:** The current "4 files / 5 acceptance criteria" rule in SKILL.md Step 3 is substrate-blind: it can't distinguish four trivial config edits (easy) from one risky core-logic change (hard). Empirically the multi-file inflection happens at ≥3 files in benchmarks, and acceptance-criteria thresholds are not validated at all.
- **Change:** In `.claude/skills/ticket-workflow/SKILL.md` Step 3 ("Assess complexity"), replace the "4+ files or 5+ acceptance criteria" rule with a two-layer policy: (1) populate `complexity` (1–10) in frontmatter via the 8-factor weighted rubric: files affected (20%), dependency count (15%), testing complexity (15%), risk level (15%), new vs modify (10%), cross-cutting concerns (10%), external API integration (5%), database changes (10%); (2) two hard triggers that force decomposition regardless of score: (a) ≥3 real files (excluding docs/lockfile-only changes) for "capable-but-limited" models, ≥5 for frontier models; (b) execution stall after ~25 meaningful tool rounds without convergence — abort and decompose. Add operational rule for current default model tier (frontier): "Attempt directly up to complexity 7 if scope is cohesive; decompose at complexity ≥8, ≥5 real files, or any high-risk cross-cutting/data/integration work." Create new file `.claude/skills/ticket-workflow/references/complexity-scoring.md` containing the full 8-factor rubric with sub-score guidance for each factor (file ranges, test scope mappings). Update each template in `references/templates.md` to include a comment beside `complexity:` field pointing to this rubric file.
- **Success criteria:**
  - `grep -c '4+ files or 5+ acceptance criteria' .claude/skills/ticket-workflow/SKILL.md` returns 0.
  - `.claude/skills/ticket-workflow/references/complexity-scoring.md` exists, lists all 8 factors with their weights summing to 100%.
  - `grep -c 'complexity ≥8' .claude/skills/ticket-workflow/SKILL.md` returns ≥1; `grep -c '25 meaningful' .claude/skills/ticket-workflow/SKILL.md` returns ≥1.
- **Research basis:** R3 § "Updated decomposition guidance" recommends the two-layer policy explicitly; the 8-factor weighted model is from `qazuor/claude-code-task-master`'s `complexity-scorer` skill (cited in R3 § "Complexity scoring approaches"); the 25-round stall trigger is from R3 § "Failure mode taxonomy" citing the SWE-bench Verified failure analysis.
- **Open questions:** (1) Which model tier defaults to which threshold — recommendation: SKILL.md states the default for the current frontier-class default model and notes the Tier A thresholds for users on Haiku-class models. (2) Whether `complexity` becomes mandatory or stays optional — recommendation: keep optional but add it to required frontmatter for any FEAT/REFAC > 1 file.

---

### D. Add structured `## Outcome` section to closing tickets and lightweight retrieval CLI

- **Type:** FEAT
- **Priority:** medium
- **Problem:** The `_archive/` directory is write-only — no future agent can query past decisions, invariants, or regression risks. As epics accumulate, structured implementation context becomes invisible to PR review and new-epic planning, exactly where it's most valuable.
- **Change:** (1) In `.claude/skills/ticket-workflow/SKILL.md` Step 7 ("Mark done"), add a sub-step: "Before the dedicated `mark ticket as done` commit, append an `## Outcome` section to the ticket using the schema in `references/outcome-schema.md`." (2) Create `.claude/skills/ticket-workflow/references/outcome-schema.md` containing the 7-subsection schema verbatim from R4 § "## Outcome schema" (Summary 2–3 sentences; Key decisions; Constraints & invariants discovered; Implementation notes — touch points + pattern; Verification; Risk/regression surface; Retrieval tags 5–10 keywords). Set length target: 120–200 words, ≤8 short bullets. Include the filled "request-scoped cache" example from R4. (3) Create `.claude/skills/ticket-workflow/scripts/archive-search.sh` implementing the two-lane retrieval from R4: Lane A is `rg -l --type md` over `.tickets/_archive/` with `--frontmatter` and `--outcome` flags filtering on YAML keys (`type`, `complexity`, `tags`) and the `## Outcome` section respectively, returning paths + matched section snippets only; Lane B is an optional `--semantic` flag that builds an in-session sentence-transformers embedding index over Outcome blocks at query time (no persisted index — built per call) and returns top-3 cosine matches. Document usage in `references/outcome-schema.md`. (4) Add to `SKILL.md` a "Querying Past Work" section: "Before creating a new epic touching unfamiliar subsystems, run `bash .claude/skills/ticket-workflow/scripts/archive-search.sh --semantic '<short epic pitch>'` and inject top-3 results' Outcome blocks into the new epic's Context section."
- **Success criteria:**
  - `.claude/skills/ticket-workflow/references/outcome-schema.md` exists, lists exactly 7 subsections in the documented order.
  - `bash .claude/skills/ticket-workflow/scripts/archive-search.sh --help` prints usage including both lanes.
  - `bash .claude/skills/ticket-workflow/scripts/archive-search.sh "test"` against an archive containing one ticket with an `## Outcome` returns a match without error.
  - SKILL.md Step 7 references the Outcome schema by file path.
- **Research basis:** R4 § "Structured outcome section proposal" provides the full 7-subsection schema with exact target length and a filled example; § "Recommendation: what to build in under 4 hours" specifies the two-lane retrieval design (ripgrep + on-demand embeddings); § "Cross-session persistence patterns" justifies the just-in-time injection model over always-on retrieval.
- **Open questions:** (1) Whether the closure ticket retroactively backfills Outcome sections for prior tickets in the epic, or only the closure ticket itself gets one — recommendation: each ticket writes its own Outcome at Step 7, closure ticket only handles archive move. (2) Which sentence-transformers model to default to — recommendation: `all-MiniLM-L6-v2` (small, CPU-fast, common); document as overridable.

---

### E. Add worktree failure recovery runbook with linked SKILL.md callouts

- **Type:** FEAT
- **Priority:** medium
- **Problem:** The worktree model is sound but agents can leave the repo in hard-to-recover states: orphaned branches from race-condition `git worktree add`, stale registrations after `rm -rf` cleanup, half-archived epics from closure ticket failures, lockfile contention from concurrent agents. Without a runbook, agents either improvise (and risk `git clean -fd` data loss) or escalate everything.
- **Change:** (1) Create `docs/runbooks/worktree-recovery.md` containing the full R5 runbook: the 10-row diagnostic command table (commands like `git worktree list --porcelain`, `git rev-parse --git-path index.lock`, `git worktree prune --dry-run`); the four named recovery procedures (Sub-agent killed mid-commit; Closure ticket partial execution; Worktree in inconsistent state — covering stale registration, broken linkage, "branch in use"; Epic branch significantly diverged from main); the six prevention conventions (always `git worktree remove`, never `rm -rf`; serialize `git worktree add` per repo with retry+jitter on `.git/config.lock` errors; use `git rev-parse --git-path` for internal paths; guard against agents writing to primary clone; prefer `git restore --staged --worktree` over `git clean -fd`; use `git worktree repair` for moved trees). (2) In `.claude/skills/ticket-workflow/SKILL.md`, add exactly two callouts at the highest-risk choke points: at the top of the "Worktree Rules" section: "**Recovery:** If any `git worktree` step fails, stop and follow `docs/runbooks/worktree-recovery.md` § Diagnostic Commands before retrying." At the top of the "Epic Closure Ticket" section: "**Recovery:** If closure fails mid-run, follow `docs/runbooks/worktree-recovery.md` § Closure Ticket Partial Execution (Goal A is the safe default)." (3) Add to "Worktree Rules" the single prevention rule that prevents the most common race: "Orchestrator must serialize `git worktree add` calls per repo. Concurrent invocations contend for `.git/config.lock` and leave orphaned branches."
- **Success criteria:**
  - `docs/runbooks/worktree-recovery.md` exists, contains a markdown table with ≥10 diagnostic commands and four `###` recovery procedure headings matching the four named failure types.
  - `grep -c 'docs/runbooks/worktree-recovery.md' .claude/skills/ticket-workflow/SKILL.md` returns exactly 2.
  - SKILL.md "Worktree Rules" mentions serialization of `git worktree add`.
- **Research basis:** R5 § "Diagnostic commands" provides the 10-row reference table; § "Recovery procedures by failure type" provides four labeled procedures with diagnose → stabilize → repair → verify steps; § "Prevention conventions" provides the six rules; § "Documentation placement recommendation" explicitly advocates a separate runbook with two minimal callouts at the highest-risk choke points (worktree creation and closure).

---

## Part 3: Implementation sequence

Implement in this order: **(1) Add TASK template** (Part 1, fix #2); **(2) Specify the orchestrator review-and-merge protocol** (Part 2, A); **(3) Specify the PR review agent protocol and rejection re-entry loop** (Part 2, B). The TASK template is the keystone unblocker — both the orchestrator's `CORRECTIVE_SUB_TICKET` escalation path and the PR review agent's ticket-materialization loop emit TASK files, so neither downstream improvement is implementable until the template exists. The orchestrator review protocol must come before the PR review agent because the orchestrator is the first quality gate and defines the merge-to-epic semantics that the PR review agent's rejection loop assumes; building the terminal gate before the upstream gate would force the PR review agent to clean up garbage the orchestrator should have caught. After these three, defects #1, #4, #5 are quick parallel chores; complexity scoring (C), Outcome retrieval (D), and worktree runbook (E) follow as additive reliability layers.
