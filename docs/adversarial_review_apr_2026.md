# Adversarial Review: Ticket Workflow System
**Date:** April 15, 2026  
**Scope:** Full system audit from initial commit (March 26, 2026) to current state  
**Artifacts reviewed:** `.claude/skills/ticket-workflow/SKILL.md` (two versions), `references/templates.md`, `docs/agentic_ticket_templates_for_ai_coding_agents.md`, original AGENTS.md + CLAUDE.md, original tickets FEAT-001 and REFAC-001, orchestration prompt `.prompts/execute-skill-tickets.md`

---

## What's Actually Good

Several design decisions in this system are genuinely well-reasoned and hold up under scrutiny.

- **Frontmatter + markdown** is the right interchange format. All major agent tools feed file contents to an LLM, so well-structured markdown is inherently portable without tool-specific parsing.
- **Verification commands as a first-class section** is correct. Without runnable commands, "done" is meaningless — agents have no feedback loop.
- **The deterministic execution protocol** (read → check deps → assess → decompose → execute → verify → mark done) is sound. Rigid step order prevents agents from self-orchestrating into analysis loops.
- **Hex IDs for epics** is the right solution to concurrent-creation collisions. Random hex is collision-resistant without requiring a coordination server.
- **Constraints section** is underrated. Explicit "Do NOT" instructions measurably reduce scope creep. Every ticket should have one.
- **200-line ticket limit** is a practical signal. Tickets requiring more than 200 lines to specify are almost certainly too large.
- **Templates embedded in the SKILL.md** is the correct portability tradeoff. Symlink the skill directory to any workspace and templates are immediately available — no per-workspace recreation required.
- **Worktree-based concurrency** is the right isolation model for parallel agent execution. Epic branch → sub-ticket worktrees → orchestrator review → epic branch merge → PR is a sound pipeline.
- **Dropping `review` status** is a deliberate architectural choice: the PR review agent is the quality gate, not per-ticket human review. Human input is concentrated at definition/specification time, not at execution time. This is coherent with a fully-agentic development model.

---

## Resolved Items (Documented for Record)

These were flagged in the initial review but are intentional design decisions.

| Item | Resolution |
|---|---|
| AGENTS.md deleted | Intentional — stale, being recreated fresh |
| `review` status removed | Intentional — PR review agent is the system's quality gate |
| Templates moved into SKILL.md | Intentional — portability via symlink; standalone template files required per-workspace recreation |
| Worktree model complexity | Intentional — required for parallel multi-agent execution |

---

## Active Defects

### 1. `docs/INDEX.md` Is a Phantom Reference

**Severity: High.**

The Epic Closure Ticket spec (SKILL.md) includes:
> **Update `docs/INDEX.md`** — remove this epic's entries from "Active Initiative Artifacts" if present.

This was accidentally pulled in from a specific workspace and does not belong in the general-purpose SKILL.md. The SKILL.md is intended to be workspace-agnostic. A reference to a file with a specific schema (`Active Initiative Artifacts`) that may not exist in a given workspace makes the closure ticket partially unimplementable in any workspace that hasn't already created this file.

**Fix:** Remove step 4 (`Update docs/INDEX.md`) from the closure ticket spec. The archive itself is the record of completion; no external index is needed for the general case.

---

### 2. No TASK Template

**Severity: Medium.**

The SKILL.md documents `TASK` as a valid ticket type for agent-generated sub-tickets, but `references/templates.md` has no TASK template. Agents decomposing complex tickets must improvise the format, leading to inconsistent sub-ticket quality. This undercuts the system's core value: structured, consistent work units.

The TASK template should be leaner than FEAT or BUG (no Context section, fewer optional fields) since TASK tickets are decomposed in-session and don't need to stand alone for long.

---

### 3. The Orchestrator's Review Step Is Unspecified

**Severity: Medium.**

The worktree model describes sub-agents working in isolated branches and an orchestrator merging their work into the epic branch. But the SKILL.md does not specify what the orchestrator actually does when reviewing a sub-agent's completed worktree before merging.

Currently, "review" here is implicit. Questions left unanswered:
- Does the orchestrator re-run verification commands, or trust the sub-agent's result?
- Does it check diff size or scope against the ticket's Constraints section?
- If the sub-agent's work is incorrect or out of scope, what does the orchestrator do? Revert and re-assign? Create a new ticket? Flag for the PR review agent?
- Does the orchestrator merge automatically if verification passed, or always pause to inspect?

The orchestrator is described as creating epics and orchestrating execution, but the review-and-merge loop — the orchestrator's most consequential action — has no protocol.

---

### 4. Locally Scoped IDs Have Unresolved Cross-Reference Ambiguity

**Severity: Low-Medium.**

Sub-ticket IDs are scoped within their epic (`FEAT-001` exists in every epic). The full reference is `EPIC-<hex>/FEAT-001`. This is documented for cross-epic references, but the `dependencies` frontmatter field is underspecified: when a ticket inside `EPIC-a7f3` lists `dependencies: [FEAT-002]`, does that mean `EPIC-a7f3/FEAT-002` or some other epic's `FEAT-002`?

The SKILL.md assumes within-epic dependencies are the common case, but the format doesn't enforce this. An agent resolving dependencies against an ambiguous ID may read the wrong ticket without knowing it.

**Fix:** Specify that bare IDs in `dependencies` always resolve within the current epic. Cross-epic dependencies must use the full `EPIC-<hex>/TYPE-NNN` form.

---

### 5. ID Assignment Is Raceable Under Concurrent Creation

**Severity: Low-Medium.**

The sub-ticket ID assignment command is a read-then-write pattern:
```bash
ls .tickets/EPIC-*_*/[A-Z]*.md | sed 's|.*/||;s|_.*||' | ...
```

Two concurrent agents both reading "next is FEAT-003" will both create `FEAT-003_*.md` in their respective worktrees. The collision surfaces only at epic branch merge time, not at creation time. The worktree model reduces this significantly (agents work in isolated branches), but it doesn't eliminate it during the ticket-creation phase before worktrees are set up.

This is low-frequency in practice. Worth tracking but not urgent.

---

### 6. Closure Ticket Steps Lack Idempotency Guards

**Severity: Low.**

The epic closure ticket runs steps sequentially: mark done, archive folder, delete orchestration prompt, clean up worktrees, commit. If this ticket fails partway through (e.g., `git mv` fails due to a merge conflict), the epic is in a half-archived state. Re-running the closure ticket would then fail on already-completed steps.

Each step should be safe to re-run: check whether the archive already exists before `git mv`, check whether the orchestration prompt still exists before `git rm`, etc. This is a minor implementation concern but worth documenting in the closure ticket spec.

---

### 7. Two SKILL.md Copies Will Diverge Without a Canonical Source

**Severity: Low.**

The skill currently lives at two paths:
1. `~/.agents/skills/ticket-workflow/SKILL.md` (global, cross-workspace)
2. `<workspace>/.claude/skills/ticket-workflow/SKILL.md` (local)

These are currently identical. The intended model is that the global location is canonical and workspaces symlink to it — but this is not documented anywhere in the SKILL.md itself. If an agent updates the local copy via a ticket (e.g., a CHORE to improve the SKILL.md), the global copy becomes stale silently.

**Fix:** Document in the SKILL.md header that the canonical source is the global path and workspace copies should be symlinks, not independent files.

---

## Research Directions

The following are the highest-value research directions for improving this system. MCP is intentionally excluded: the SKILL.md approach is more portable (symlink vs. per-workspace server configuration), lighter (no process to maintain), and the primary problem MCP would solve — ID generation races — is already largely contained by the worktree isolation model. The remaining coordination problems are better addressed by a lightweight git hook or shell script than a full server.

---

### R1: PR Review Agent Design

**Priority: High — this is the system's quality gate and is currently unspecified.**

The fully-agentic model places the PR review agent as the terminal quality checkpoint before code reaches main. All execution agent output flows through it. But the current SKILL.md says nothing about how this agent should work, what it's checking for, or how its outputs feed back into the ticket system.

This is the most consequential unspecified component in the system. If the PR review agent is weak, the entire pipeline's quality guarantee is weak.

**Research questions:**
- What should a PR review agent check that execution agents and orchestrators don't already check? (Scope adherence, cross-ticket consistency, test coverage, security patterns, architectural fit?)
- How does it communicate rejection back into the ticket system? Options: new corrective tickets, PR comments, re-opening closed tickets, direct orchestrator notification?
- What is the feedback loop when the PR review agent rejects? Does a new epic get created? Does the existing epic reopen?
- How do existing agentic PR review systems (GitHub Copilot code review, Cursor's review mode, Graphite's Aviator) handle rejection feedback?
- Should the PR review agent have read access to `_archive/` to check for regressions against prior completed epics?
- Is one PR review agent sufficient, or should there be staged review (e.g., automated checks first, then a higher-capability agent for architecture review)?

**Suggested output:** Research doc + PR review agent protocol spec (what it checks, how it signals rejection, how rejections re-enter the ticket system).

---

### R2: Orchestrator Review Protocol and Escalation Paths

**Priority: High — the orchestrator's review loop has no protocol.**

The SKILL.md describes an orchestrator that creates epics, assigns sub-tickets, and eventually creates a PR. But the most critical orchestrator action — reviewing each sub-agent's completed worktree and deciding whether to merge it to the epic branch — is entirely unspecified.

This gap matters because the orchestrator review is the first quality gate. Code that passes it reaches the PR review agent; code that fails it should loop back to execution. Without a protocol, orchestrators either auto-merge everything (no review) or block indefinitely waiting for human instruction.

**Research questions:**
- What should the orchestrator verify before merging a sub-ticket's worktree to the epic branch? (Verification commands re-run? Diff against ticket scope? Constraint section compliance?)
- When a sub-agent's work fails the orchestrator's review, what are the escalation options? (Re-assign the ticket, create a corrective TASK ticket, mark blocked and escalate to human, revert and re-execute from scratch?)
- How does the orchestrator handle partial failures — where some sub-tickets pass review and others fail? Can it merge the passing ones and hold the failing ones, or must the epic be atomic?
- What do orchestration frameworks (LangGraph, CrewAI, Prefect AI) use as standard patterns for review-and-escalation loops?
- How should the orchestration prompt (`.prompts/orchestration/`) be structured to encode the review protocol without duplicating the SKILL.md?

**Suggested output:** Research doc + orchestrator review protocol spec + updated orchestration prompt template.

---

### R3: Agent Failure Modes and Decomposition Calibration

**Priority: High — the decomposition threshold is the system's primary reliability lever and it's empirically ungrounded.**

The threshold "decompose if 4+ files or 5+ acceptance criteria" drives every execution decision. It's a reasonable default, but it's not derived from data. A single-file change that coordinates 5 external systems is harder than a 4-file change with no dependencies. The threshold is substrate-blind.

More importantly, as models improve (Claude 4's 200K context vs. older models' 32K), the right threshold changes. The system should either acknowledge that the threshold is tunable or derive a better signal.

**Research questions:**
- What properties of a task actually predict agent failure? (File count, AC count, external system count, dependency graph depth, ticket age, ticket type?)
- Does the "4 files / 5 ACs" threshold correspond to a real inflection point in agent reliability, or is it arbitrary?
- How does Task Master AI's complexity scoring (1-10) relate to actual agent execution success rates? Is complexity a better signal than threshold rules?
- What's the cost asymmetry: over-decomposing (too many sub-tickets, administrative overhead) vs. under-decomposing (agent takes on too much, fails partway)?
- How have practitioners adjusted decomposition rules as model context windows expanded? Is there a "right" decomposition granularity that's model-independent?

**Suggested output:** Research doc + updated decomposition heuristics grounded in empirical data, with explicit guidance on tuning for different model capability levels.

---

### R4: Archive as Active Knowledge Base

**Priority: Medium — the archive's value grows with each epic and is currently unreachable.**

The `_archive/` directory is write-only. Completed epics go in; nothing comes out. As the system matures — more epics, more sub-tickets, more implementation decisions recorded in ticket files — the archive accumulates structured, version-controlled context that future agents can't access.

This matters most for the PR review agent (which could use archive context to detect regressions against prior patterns) and for new epic creation (which could surface relevant prior art). An archive that's queryable, even crudely, is significantly more valuable than one that isn't.

**Research questions:**
- What retrieval mechanism is appropriate for this scale? Simple `grep` conventions, a lightweight semantic search script, or an embedding-based store?
- At what granularity is retrieval most useful — epic level (what was the goal?), ticket level (what was the implementation approach?), or commit level (what specifically changed)?
- How do similar systems (Task Master AI, Shrimp Task Manager, Claude's memory systems) handle cross-session context persistence?
- Is there a convention for writing ticket "implementation notes" at close time that would make future retrieval more useful (e.g., a structured `## Outcome` section in closed tickets)?
- When should an agent proactively query the archive vs. when should it be injected automatically (e.g., always injected when creating a new epic)?

**Suggested output:** Research doc + proposed retrieval convention (likely grep-based or semantic) + recommendation on whether tickets should include a structured outcome section at close time.

---

### R5: Worktree Failure Recovery Runbook

**Priority: Medium — the worktree model is the right design but has no documented failure recovery.**

The worktree model is sophisticated and correct for parallel execution. But agents operating in worktrees can fail in ways that leave the repository in hard-to-recover states: uncommitted changes in a dead worktree, half-executed closure tickets, orphaned branches from agents that drifted from the naming convention, or epic branches with merge conflicts that block the PR.

Unlike the other research directions, this isn't primarily a research question — it's a protocol and documentation gap. But it requires surveying real worktree failure modes before the runbook can be written.

**Research questions:**
- What are the most common git worktree failure modes in multi-agent development? (Based on practitioner reports, GitHub issues, or experimental data)
- What is the recovery procedure for each: (a) agent killed mid-commit, (b) closure ticket fails after partially archiving, (c) worktree left checked out on a branch, (d) epic branch diverged from main significantly before PR?
- Is there a safe "force reset" procedure that can clean up a failed worktree without risking loss of good sub-ticket work?
- How should the SKILL.md document these recovery paths without making the happy-path protocol harder to follow?

**Suggested output:** Research doc + worktree failure recovery runbook, formatted for direct inclusion in the SKILL.md or as a linked reference document.

---

## Priority Summary

| Research Direction | Priority | Key Output |
|---|---|---|
| R1: PR Review Agent Design | High | Review agent protocol spec |
| R2: Orchestrator Review Protocol | High | Orchestrator loop spec + prompt template |
| R3: Failure Modes and Decomposition | High | Updated decomposition heuristics |
| R4: Archive as Knowledge Base | Medium | Retrieval convention + outcome section proposal |
| R5: Worktree Failure Recovery | Medium | Recovery runbook |

---

## Immediate Fixes (No Research Needed)

These are clear defects with known solutions that should be addressed before research begins:

1. **Remove `docs/INDEX.md` from the closure ticket spec** — workspace-specific artifact accidentally included in a general-purpose skill. Remove step 4 from the Epic Closure Ticket section of the SKILL.md.

2. **Add a TASK template to `references/templates.md`** — agent-generated sub-tickets need a template. Lean format: fewer optional fields, no Context section, ≤3 ACs, explicit `parent` required.

3. **Clarify `dependencies` ID resolution scope** — specify in the SKILL.md that bare IDs in `dependencies` resolve within the current epic; cross-epic references must use `EPIC-<hex>/TYPE-NNN`.

4. **Add idempotency guards to the closure ticket spec** — document the "check before acting" pattern for each step (check archive exists before `git mv`, check prompt exists before `git rm`, etc.).

5. **Document canonical SKILL.md location** — specify in the SKILL.md header that the canonical source is `~/.agents/skills/ticket-workflow/` and workspace copies should be symlinks.

6. **Recreate AGENTS.md** — lean, workspace-agnostic entry point that tells agents the ticket system exists and points to the SKILL.md. No procedural detail in AGENTS.md — that lives in the skill.
