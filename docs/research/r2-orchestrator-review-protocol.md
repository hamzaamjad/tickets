# R2 Orchestrator Review Protocol and Escalation Paths

## Orchestrator review checklist

A deterministic orchestrator review should behave like a lightweight, tool-assisted code inspection: it exists to catch *omissions*, *scope creep*, and *silent behavior changes* that tests can miss. Classic research and practitioner guidance both emphasize that structured review improves outcomes (and that checklists specifically reduce “missing things” defects). citeturn15view0turn15view1turn3view1turn3view0

Below is an ordered, “prompt-encodable” checklist. Each step has a concrete pass/fail criterion and a reason it catches real problems.

1. **Confirm the review inputs are complete and consistent (entry criteria).**  
   **Check:** ticket scope + constraints (“Do NOT…”) + acceptance criteria + declared file-path allowlist + verification log are all present; the sub-agent marked done; the worktree has a branch name that matches the ticket; and there is a clean commit history for the ticket’s work (no half-done changes).  
   **Why:** Formal inspections and modern review both rely on explicit entry/exit criteria to avoid reviewing garbage or incomplete artifacts; missing context is a leading cause of superficial review. citeturn3view1turn15view0turn3view0

2. **Re-run a minimal “sanity verification” on the orchestrator machine (not just trust the log).**  
   **Check:** run the fastest subset that detects environment drift (e.g., `lint + unit` or the project’s “smoke” script) and ensure results match the provided verification log (same commands, same pass).  
   **Why:** Worktrees isolate workspace state but not all shared repo/global assumptions (hooks, shared config), and logs can be stale or misreported; rerunning a small subset reduces “it passed over there” failures. Worktrees are multiple working directories tied to the same repo, and you explicitly remove them when done—treat the worktree boundary as a gate where you re-check the truth. citeturn6view2turn9view0turn15view0

3. **Compute diff stats and classify review size vs. expected ticket size.**  
   **Check:** count files changed and LOC changed; compare to the ticket’s declared scope and typical thresholds (e.g., flag >400 LOC for “high scrutiny” and require explicit justification for “why this much change”).  
   **Why:** Multiple empirical/practitioner sources show defect-finding drops as reviewed code volume rises; SmartBear’s Cisco-team study argues effectiveness diminishes beyond ~200–400 LOC per review chunk, and recommends time-boxing and smaller review units. Use size as a *risk multiplier*, not an auto-fail. citeturn15view0

4. **Verify “changed files” stay inside the ticket’s declared file-path allowlist (side-effects gate).**  
   **Check:** `changed_paths ⊆ allowlist` (with a small, explicitly allowed “exceptions list” such as generated lockfiles or shared configs if the ticket allows them). If there are out-of-scope file edits, require (a) an explicit rationale tied to acceptance criteria, and (b) a compatibility check with the epic branch.  
   **Why:** This is the single most deterministic defense against “agent wandered into unrelated refactors.” In multi-agent workflows, conflicts are often deferred to merge time; catching cross-area edits early prevents merge storms later. citeturn9view0turn15view3

5. **Hard-check the ticket “Constraints” section as non-negotiable policy.**  
   **Check:** for each “Do NOT” rule, map to an observable test:  
   *If “Do NOT change public API” → check exported symbols / routes / schemas weren’t altered.*  
   *If “Do NOT add dependencies” → check lockfile / manifest diff is empty.*  
   *If “Do NOT touch migrations” → check migrations directory unchanged.*  
   Any violation is an automatic review fail unless the ticket itself explicitly amended the constraint.  
   **Why:** Constraints are effectively organizational policy boundaries. Recent security-oriented workflow research argues probabilistic “guardrails” are routinely bypassed; the safest approach is deterministic policy enforcement at boundaries. citeturn18view0turn5view2

6. **Validate acceptance criteria by building a criteria-to-evidence matrix (not “tests passed”).**  
   **Check:** for each acceptance criterion, write one line of evidence: *which code path implements it, and which test/log demonstrates it.* If any criterion has no evidence, fail review.  
   **Why:** Modern code review is as much about change understanding as defect hunting; studies highlight that reviews often miss deeper design/intent issues and that understanding is key. A matrix forces intent alignment. citeturn3view0turn4view0

7. **Check “negative space”: what *should* have changed but didn’t (omissions gate).**  
   **Check:** given scope + acceptance criteria, ask:  
   *Should there be at least one new/modified test? docs? type updates? telemetry?*  
   Require explicit justification when expected artifacts are missing.  
   **Why:** SmartBear explicitly calls out omissions as hard-to-find defects and recommends checklists to counter them; a second-pass orchestrator is primarily an “omission hunter.” citeturn15view0turn15view1turn3view1

8. **Spot-check maintainability and code health, but don’t block on nits.**  
   **Check:** evaluate naming/readability, duplication, and whether the change worsens architectural boundaries. Block only on maintainability regressions that create future risk, not stylistic preferences (unless violating a standard).  
   **Why:** Google’s standard frames review as ensuring code health improves over time, balancing progress with quality; it explicitly warns against seeking perfection or blocking for minor polish. citeturn4view0

9. **Search for “implicit behavior changes” in shared or configuration surfaces.**  
   **Check:** if diff touches config files, shared utilities, auth/session, data models, or build scripts, require:  
   - explicit backward compatibility note  
   - at least one targeted test or integration check  
   - a short “blast radius” statement  
   **Why:** These are high-leverage files where small diffs have wide impact; modern review research shows reviews often devolve into minor issues unless reviewers deliberately focus on deeper implications. citeturn3view0turn3view1

10. **Assess mergeability against the current epic branch (partial-merge risk control).**  
   **Check:** rebase/merge the ticket branch onto the latest epic branch in the worktree (or a temporary integration worktree) and confirm:  
   - conflicts are resolved (or none exist)  
   - the minimal sanity suite still passes post-rebase  
   **Why:** Git will surface conflicts when divergent commits touch the same lines; rebasing can introduce conflicts and forces explicit resolution. This step prevents “passes alone, breaks when combined.” citeturn15view3turn15view2turn9view0

11. **Handle partial epic failures with explicit dependency rules.**  
   **Check:** only merge a passing ticket if (a) its dependencies are already merged, or (b) it is dependency-free / guarded (feature flag, non-reachable code path, or purely additive API that nothing calls yet). Otherwise, hold it to avoid integrating code that can’t compile/run without the missing dependency ticket.  
   **Why:** Workflow orchestrators treat tasks as dependency-ordered units; similarly, the epic should behave like a dependency graph, not a pile of independent patches. Prefect explicitly frames tasks as units that can run concurrently with dependency semantics and state-driven orchestration; apply the same idea to merge ordering. citeturn6view3turn10view0turn15view3

12. **Finalize decision with a strict, logged outcome: MERGE, FIX, REASSIGN, ESCALATE, or RESTART.**  
   **Check:** produce a short decision record containing: scope delta summary, constraints pass/fail, acceptance criteria mapping status, and the chosen escalation (if any).  
   **Why:** Without an explicit exit record, orchestrators either auto-merge or stall. Modern review research emphasizes that lightweight processes lack mandated criteria; your protocol fixes that by making the outcome auditable and repeatable. citeturn3view1turn15view1

## Escalation pattern comparison

A practical escalation system needs two things: (1) a failure taxonomy that maps cleanly to actions, and (2) a retry budget to prevent infinite loops (a known failure mode in workflow automation). Temporal and Prefect communities document how default retries/automations can effectively run “forever” or re-run indefinitely when misapplied—your orchestrator should avoid reproducing that pathology in the code-review layer. citeturn11view0turn12view0turn5view5turn5view6

### Re-assign the same ticket to a new sub-agent

**Best when:** the failure suggests *agent capability mismatch* or *process corruption*, not a small patch defect. Typical triggers:
- The implementation repeatedly ignores constraints or scope (e.g., keeps editing out-of-allowlist paths after being corrected once).  
- The change is conceptually wrong (misread requirement), and fixing it would be equivalent to re-implementing.  
- The branch/worktree is chaotic (many unrelated commits, unclear intent), making targeted correction slower than redo.

**Why this works:** It is the “retry with a fresh worker” pattern from durable execution systems: if you think the execution environment or decision path is bad, restarting can be cheaper than debugging. In Temporal, retries are appropriate for transient failures but you also explicitly mark errors as non-retryable when they won’t improve with repetition; reassigning is the human-free analog—try again only when you expect a different outcome. citeturn5view5turn5view6

**Downstream impact:** tends to increase cycle time (you pay the full implement cost again) but can improve quality if the first attempt indicates systematic misunderstanding.

### Create a corrective sub-ticket targeting the specific failure

**Best when:** the work is mostly correct and the failure is *localized* and *well-scoped*, such as:
- Missing/weak tests for one acceptance criterion  
- Minor constraint-adjacent issue that can be fixed without redesign (e.g., touched one extra file that can be reverted)  
- Documentation/telemetry missing but easy to add  
- Small refactor needed for readability/maintainability

**Why this works:** It mirrors “guardrails + feedback loop” patterns in agent frameworks: validate output, and if it fails, return precise feedback for a bounded correction. CrewAI explicitly supports task guardrails that validate outputs before passing downstream, including deterministic function-based checks; use the same idea for code deltas—create a correction unit that is deterministic and narrow. citeturn5view2turn24view0

**Downstream impact:** usually fastest path to epic completion because you salvage the good work and isolate the fix. Cost: more coordination and more merge sequencing complexity (you now have an extra ticket that may need to land before/after the original).

### Mark the ticket blocked and escalate to the downstream PR review agent with a flag

**Best when:** the failure is *high-stakes or ambiguous*, where a “stronger reviewer” (or a different review stage) is appropriate:
- Design/API judgment calls not resolvable from ticket text  
- Security/privacy concerns that require deeper threat modeling  
- Cross-cutting architecture changes touching shared surfaces  
- Conflicts between constraints and acceptance criteria (spec inconsistency)

**Why this works:** Many orchestration systems embed “human-in-the-loop” or “approval workflows” as explicit escalation nodes. LangGraph documents multi-level approvals and review/edit workflows where execution pauses until an approver decides; Temporal likewise highlights human-in-the-loop as a common design pattern for steps that need human judgment or external intervention. Your PR review agent is the next “approval node” in this pipeline. citeturn8view0turn20view2turn5view0

**Downstream impact:** preserves progress while making risk explicit. However, if escalations become frequent, your orchestrator layer becomes toothless—track rates and push recurring ambiguity back into better ticket specs.

### Revert the sub-agent’s branch and restart from scratch

**Best when:** the change introduces *systemic risk* or *unrecoverable divergence*:
- Major constraint violations with broad impact (e.g., altered public API when forbidden)  
- Large diff far outside scope, implying the agent “went rogue”  
- Hard-to-trust state: generated files, sweeping refactors, or unexplained dependency changes  
- Fixing would require large-scale manual patch surgery, risking new defects

**Why this works:** This is the “compensation/rollback” idea from workflow engines: if a multi-step process fails in a way that threatens system integrity, you undo and restart. Temporal’s saga pattern explicitly frames compensating actions as the way to restore consistency when part of a coordinated sequence fails. Treat reverting as your compensation step for an unsafe patch. citeturn5view5turn0search3turn0search7

**Downstream impact:** slowest option in elapsed time, but often best for quality when trust is lost. To avoid thrash, enforce a restart threshold (e.g., “restart only after a hard policy breach or >2 failed correction cycles”).

## Framework pattern survey

Agent orchestration frameworks and workflow engines converge on a small set of production-reliable patterns: explicit state machines, validation gates, termination conditions, bounded retries, and compensation. What differs is *where* the loop runs (LLM conversation vs. durable workflow vs. code integration) and *how deterministic* the checks are.

image_group{"layout":"carousel","aspect_ratio":"16:9","query":["LangGraph workflow graph diagram","Temporal workflow diagram durable execution","Prefect flow graph tasks states diagram","git worktree diagram multiple worktrees"],"num_per_query":1}

### LangGraph

LangGraph makes review/escalation explicit via interrupts: graphs can pause before/after nodes, persist state, and resume once an external actor approves or edits. It also documents “review and edit workflows” and a “retry after human review” pattern, plus a caution that looped interrupt logic re-executes nodes from the start. citeturn5view0turn8view0turn13view1

**Transferable pattern to git worktrees:** treat “merge” as the high-stakes node. Your orchestrator is effectively an interrupt/approval node: it inspects diff + evidence, then either resumes (merge) or routes back (correction/reassign). The “re-exec from start” warning maps cleanly to your need for retry budgets—don’t endlessly send vague “fix it” feedback. citeturn13view1turn15view0

### CrewAI

CrewAI’s hierarchical process is built around a manager agent that delegates tasks and validates outcomes; its docs explicitly describe this manager role as coordinating and validating results. It also supports “task guardrails” that validate and transform outputs before passing them onward, including deterministic function-based guardrails versus LLM-based checks for subjective criteria. citeturn6view0turn24view0turn5view2

**Transferable pattern:** implement your orchestrator review as “function-based guardrails over the repo diff” wherever possible (path allowlist, dependency diff checks, constraint checks), and reserve LLM judgment for mapping acceptance criteria to code. This hybrid mirrors production practice: deterministic gates first, probabilistic reasoning second. citeturn5view2turn18view0

**Known failure mode:** if you rely on LLM-based guardrails for hard policy, you get non-deterministic enforcement. CrewAI’s own distinction between function-based and LLM-based guardrails is an implicit warning: use deterministic checks for deterministic requirements. citeturn5view2turn18view0

### AutoGen

AutoGen documents a “reflection pattern” where a critic agent evaluates a primary agent, and it emphasizes explicit termination conditions (e.g., stop when the critic outputs “APPROVE”), optional external termination, and the ability to reset team state between unrelated tasks. citeturn5view3turn13view2turn13view3turn6view1

**Transferable pattern:** your orchestrator protocol should have:  
- a crisp “APPROVE/MERGE” termination criterion (all checklist items satisfied), and  
- explicit stop conditions when review is failing repeatedly (escalate or restart), plus  
- a “reset context” discipline so the orchestrator doesn’t let one ticket’s details bleed into another. citeturn13view3turn10view0

**Known failure mode:** agent loops without strong termination conditions (critic keeps asking for changes). AutoGen’s emphasis on termination conditions is the production fix; encode the same into your escalation logic. citeturn13view2turn12view0

### Prefect AI

Prefect frames flows/tasks as templates and *runs* as entities that take on states; it records state transitions and supports retries, caching, timeouts, and hooks/automations. Its agent-focused content describes durable execution: retrying a workflow can skip completed work via cached task results, and it supports granular retry policies for different failure characteristics (LLM calls vs tools). Prefect also warns that client-side hooks aren’t guaranteed and suggests automations for robust responses. citeturn10view0turn6view3turn20view1turn5view4turn12view0

**Transferable pattern:** treat “ticket review” as a state machine with explicit terminal states: MERGED, NEEDS_FIX, REASSIGNED, ESCALATED, RESTARTED. Also borrow Prefect’s core operational idea: granular retries with limits (small fix loops allowed; systemic failures route to different actions). citeturn20view1turn10view0turn12view0

**Known failure mode:** naive automations can re-run indefinitely when the underlying error is not transient (documented by users observing endless crash reruns). Your orchestrator must cap “fix cycles” and switch strategies. citeturn12view0turn15view0

### Temporal workflows

Temporal’s documentation distinguishes failure types and strongly emphasizes: retry policies for activities, marking some errors as non-retryable, idempotency, and saga-style rollback/compensation. It also warns against misapplied retries (activities can retry “practically forever” under certain default timeout setups), and frames durable execution as “crash-proof execution” where state is captured and progress resumes after failures. citeturn5view5turn5view6turn11view0turn23view1turn23view3

**Transferable pattern:**  
- classify orchestrator review failures as *retryable* (small, local fix) vs *non-retryable* (scope/constraint breach → restart or reassign). citeturn5view5turn5view6  
- use “compensation” as revert-and-restart when safety is compromised. citeturn0search3turn5view5  
- implement a retry budget to prevent infinite loops, mirroring Temporal’s need to reason about retry limits/timeouts. citeturn11view0turn5view6  

**Production reliability note:** Temporal markets and documents durable execution as a reliability abstraction used in large-scale systems; case studies describe improved reliability/productivity when adopting code-first workflow orchestration. The practical lesson for your pipeline: orchestrators should be code-driven state machines with explicit policy gates, not ad-hoc chat behavior. citeturn22view0turn23view2turn4view0

## Orchestrator prompt template

This template encodes the checklist and escalation logic as executable instructions. It is deliberately model-agnostic and avoids tool-specific APIs; you inject your tooling commands and paths per environment. Its structure also enforces separation between (a) persistent protocol and (b) per-epic/per-ticket brief—mirroring the “templates vs runs/state” separation emphasized by workflow orchestrators. citeturn10view0turn13view3turn5view0

```text
SYSTEM / ROLE
You are the Orchestrator Agent for a multi-agent software development pipeline using git worktrees.
Your job is to review completed sub-agent tickets and decide whether to merge each ticket branch into the shared epic branch.
This is the last fully automated quality gate before the downstream PR review step.

NON-NEGOTIABLES
- Follow the Review Protocol exactly, in order.
- Treat the ticket “Constraints / Do NOT” rules as hard policy.
- Do NOT merge if any constraint is violated or any acceptance criterion lacks evidence.
- Do NOT stall. If review fails, choose an escalation action deterministically.
- Keep a written Decision Record for every ticket (MERGED / NEEDS_FIX / REASSIGNED / ESCALATED / RESTARTED).

PERSISTENT ORCHESTRATOR INSTRUCTIONS (stable, always apply)
[Insert your standing conventions: branching rules, merge strategy (merge/squash/rebase), formatting standards,
test command conventions, allowed dependency policy, security checks, commit message rules, etc.]

EPIC TASK BRIEF (session-specific, injected each run)
Epic name: <EPIC_NAME>
Epic branch: <EPIC_BRANCH>
Epic goal: <EPIC_GOAL>
Epic constraints: <EPIC_LEVEL_CONSTRAINTS>
Ticket dependency graph / ordering notes: <TICKET_DEPENDENCIES_AND_ORDER>

TICKET PACKET (per ticket, injected each review)
Ticket ID: <TICKET_ID>
Ticket title: <TICKET_TITLE>
Worktree path: <WORKTREE_PATH>
Ticket branch: <TICKET_BRANCH>
Base for diff: <EPIC_BRANCH>
Scope summary: <SCOPE_SUMMARY>
Allowed file paths (allowlist): <ALLOWLIST_PATHS>
Acceptance criteria: <ACCEPTANCE_CRITERIA_LIST>
Constraints / Do NOT rules: <CONSTRAINTS_LIST>
Verification commands run by sub-agent: <VERIFICATION_COMMANDS>
Verification log output: <VERIFICATION_LOG>

REVIEW PROTOCOL (execute in order; produce a YES/NO for each gate)

Gate A — Entry completeness
A1. Confirm all Ticket Packet fields are present and non-empty.
A2. Confirm worktree is on <TICKET_BRANCH> and contains a clean, reviewable commit set for this ticket.

Gate B — Minimal independent verification
B1. Re-run the minimal “sanity suite” on your machine: <SANITY_COMMANDS>.
B2. If sanity suite fails, classify failure as: TRANSIENT (flake/env) vs CODE REGRESSION (reproducible).
    - If TRANSIENT: re-run once. If still failing, escalate per decision logic.
    - If CODE REGRESSION: fail review.

Gate C — Diff size vs scope risk classification
C1. Compute diff stats vs <EPIC_BRANCH>: files_changed, lines_added, lines_deleted.
C2. Classify size: SMALL / MEDIUM / LARGE using these thresholds: <SIZE_THRESHOLDS>.
C3. If LARGE, require explicit justification tied to acceptance criteria; otherwise fail review.

Gate D — Side-effects and path allowlist
D1. List all changed file paths.
D2. If any changed path is outside <ALLOWLIST_PATHS>:
    - If it is in <EXPLICIT_ALLOWED_EXCEPTIONS>: continue.
    - Else: fail review as SCOPE_BREACH unless the ticket packet explicitly justifies it.

Gate E — Constraints (hard policy)
For each constraint in <CONSTRAINTS_LIST>:
E1. State “PASS/FAIL” with one sentence of evidence from the diff.
E2. If any FAIL: stop review and choose escalation action (do not merge).

Gate F — Acceptance criteria evidence matrix
For each acceptance criterion:
F1. Provide an Evidence Row:
    - criterion: <text>
    - implementation pointer: file(s) + function/class/endpoint
    - verification pointer: test name/log line/manual reasoning
F2. If any criterion has missing evidence: fail review.

Gate G — Omission hunting (negative space)
G1. Based on scope + criteria, check if expected artifacts exist:
    - tests updated/added where behavior changed
    - docs/comments updated where user-facing or tricky logic changed
    - telemetry/logging updated if required by conventions
G2. If a normally expected artifact is missing, mark as NEEDS_FIX unless explicitly justified.

Gate H — Mergeability against latest epic
H1. Update <EPIC_BRANCH> to latest.
H2. Rebase/merge <TICKET_BRANCH> onto latest <EPIC_BRANCH> in the worktree.
H3. If conflicts occur:
    - If conflicts are trivial and resolvable without design decisions, resolve and continue.
    - Else fail review as INTEGRATION_CONFLICT.
H4. Re-run <SANITY_COMMANDS> post-rebase. Must pass.

Gate I — Partial epic merge rule
I1. Determine whether this ticket depends on any non-merged tickets per <TICKET_DEPENDENCIES_AND_ORDER>.
I2. If dependencies are not merged and the change is not safely isolated (feature-flagged, additive, non-reachable):
    - Mark as BLOCKED_DEPENDENCY and do not merge.

ESCALATION DECISION LOGIC (choose exactly one)

If any of these failure types occur → action:
1) CONSTRAINT_VIOLATION or major SCOPE_BREACH or unsafe dependency changes:
   → RESTART_FROM_SCRATCH (revert/close branch/worktree; create fresh ticket run)
2) ACCEPTANCE_CRITERIA_MISMATCH but diff is small-to-medium and fix is localized:
   → CORRECTIVE_SUB_TICKET (create a tight fix ticket; do not merge until fixed)
3) INTEGRATION_CONFLICT requiring design judgment OR spec ambiguity (constraints vs criteria conflict):
   → ESCALATE_TO_PR_AGENT_WITH_FLAG (summarize risk; hold merge unless policy says otherwise)
4) Repeated low-quality attempts on same ticket (>= <MAX_FIX_CYCLES>) OR chaotic branch history:
   → REASSIGN_NEW_SUB_AGENT

MERGE STEPS (only if all gates PASS and ticket is not blocked)
M1. Merge <TICKET_BRANCH> into <EPIC_BRANCH> using <MERGE_METHOD>.
M2. Run <POST_MERGE_CHECKS> on <EPIC_BRANCH>.
M3. Record merge commit SHA and summary of what changed.
M4. Clean up: archive logs, remove the worktree if your system expects it: <WORKTREE_CLEANUP_STEPS>.

OUTPUT FORMAT (always produce)
1) Review Summary:
   - diff stats
   - changed paths summary
   - size classification
2) Gate Results: A–I with PASS/FAIL
3) Acceptance Criteria Evidence Matrix
4) Decision:
   - one of: MERGED / NEEDS_FIX / REASSIGNED / ESCALATED / RESTARTED / BLOCKED_DEPENDENCY
   - rationale in 3–6 sentences
5) Next Actions:
   - exact commands or tickets you will create, and what the next agent should do
```

This template is intentionally strict about termination conditions and bounded loops—mirroring production lessons from multi-agent frameworks (explicit stop rules) and workflow engines (retry policies plus non-retryable boundaries). It also makes the orchestrator’s job “deterministic by construction”: most gates are objective checks, and the few judgment calls (acceptance criteria mapping, maintainability regressions) are forced into explicit written evidence. citeturn13view2turn5view6turn15view0turn17view0