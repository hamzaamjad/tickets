# PR Review Agent Design for a Fully Agentic SDLC

## Defect taxonomy

A PR-level review agent is most valuable where *localized* verification (ticket-scoped tests/linters) and even an orchestrator rerun still miss problems that are only visible when you: (a) see the *entire epic diff*, (b) reason about *system-level invariants*, and (c) compare changes against *established patterns and history*. Empirical software engineering research supports that code review often detects large volumes of issues that execution-based QA cannot—especially “evolvability” (maintainability/understandability) defects. citeturn9view0turn9view1turn32view0

Below is a prioritized taxonomy of defect classes that reliably surface at PR scope in practice, with why they escape earlier stages and a concrete example/source pattern for each.

**Cross-ticket contract mismatches (API/schema/event contracts) — highest frequency and highest blast radius**  
What it is: changes that are “correct” within the ticket’s boundary but break implicit or explicit contracts between modules/services/tickets: REST/GraphQL endpoints, DTOs, database schema migrations, queue payloads, feature-flag semantics, or even internal helper function semantics depended on elsewhere. citeturn29view1turn25view0turn6search0  
Why it escapes: ticket verification typically runs unit tests for the touched component and does not include downstream consumers, multi-repo dependents, or end-to-end contract tests; orchestrator rerunning the same ticket commands won’t add missing cross-system coverage. citeturn29view1turn6search0  
Concrete evidence/example: CodeRabbit explicitly positions “API contracts” and “shared libraries” as common cross-repository ripple sources and builds multi-repo analysis to detect them. citeturn29view1turn25view0 Graphite’s “review comments” taxonomy also calls out “mismatches between API usage and implementation” and “inconsistencies between code behavior and documentation” as review findings. citeturn27view0

**Architectural drift and codebase health regressions (pattern violations, layering breaks, dependency creep)**  
What it is: changes that erode architectural constraints—e.g., UI layer reaching into persistence, new dependency introduced across a boundary, “just this once” duplication, inconsistent error-handling strategy, or a new abstraction that conflicts with existing ones. These are classically *review* problems: they may never fail tests, yet permanently increase entropy. citeturn6search0turn9view0turn6search4  
Why it escapes: per-ticket tests validate behavior, not architecture. Mäntylä & Lassenius found ~75% of defects discovered in code reviews did **not** affect visible functionality; they improved “evolvability” (ease of understanding/modifying) and are not detectable by execution-based QA methods. citeturn9view0turn9view1  
Concrete evidence/example: Google’s reviewer guidance explicitly prioritizes “overall design” and whether the change “integrate[s] well with the rest of your system,” i.e., system-level architectural fit. citeturn6search0turn6search4

**Coverage gaps and “test suite lies” (missing integration tests, missing negative/edge tests, untested wiring)**  
What it is: the PR “passes” existing tests but lacks tests for new behaviors, cross-module wiring, failure modes, and edge conditions. This includes the classic symptom: feature code exists, tests pass, but nothing is wired into the request path, job scheduler, or config, so production behavior is unchanged (or breaks silently). citeturn6search0turn33view0turn7search19  
Why it escapes: ticket verification typically runs pre-existing test targets plus whatever the ticket author specified; if the ticket didn’t define coverage expectations, nothing enforces cross-ticket test adequacy. The “When Testing Meets Code Review” line of research underscores that reviews of tests and test adequacy are a distinct practice area, not automatically enforced by running tests. citeturn6search0turn1search28  
Concrete evidence/example: Graphite’s reviewer categories explicitly include “edge cases” like missing null checks, race conditions, and unexpected side effects—issues that often lack explicit tests. citeturn27view0 Practitioners building agentic review systems (e.g., HubSpot’s automated review work) report early versions were “nitpicky/verbose” and needed a second-stage judge to keep feedback actionable—implicitly: catching test gaps is valuable only if the signal is high. citeturn3view0

**Security anti-patterns that pass tests (authZ regressions, injection surfaces, crypto misuse, insecure defaults)**  
What it is: security weaknesses introduced in otherwise “working” code: missing authorization checks, unsafe string interpolation into queries, insecure crypto, broadening permissions, risky deserialization, secrets leakage via logs, etc. citeturn6search1turn6search17turn27view0turn33view0  
Why it escapes: unit tests rarely encode adversarial behavior; orchestrator rerunning unit/lint commands won’t detect missing threat modeling. Even SAST coverage is incomplete and has significant false positives/triage overhead. citeturn17view0turn18view0turn33view0  
Concrete evidence/example: an empirical study of secure code review in OpenSSL/PHP found reviewers raised security concerns across **35 of 40** CWE-699 weakness categories, demonstrating review’s broad security detection role—but also that many concerns are merely acknowledged or debated rather than fixed, implying a need for structured follow-up work. citeturn33view0 OWASP provides explicit vulnerability-focused review guidance and checklists precisely because tests do not reliably catch these categories. citeturn6search1turn6search17

**Performance/scalability regressions hidden by green CI (N+1 queries, algorithmic regressions, memory bloat)**  
What it is: code that is functionally correct but introduces avoidable performance risk: O(n²) where O(n) exists, N+1 query patterns, unbounded retries, heavyweight allocations in hot paths, unnecessary network calls, or cache stampedes. citeturn27view0turn6search0  
Why it escapes: most ticket-level verification doesn’t include performance regression tests or production-like load profiles; performance issues rarely fail unit tests and often won’t show up until integrated behavior or real traffic. citeturn27view0turn6search0  
Concrete evidence/example: Graphite explicitly lists “performance issues” and “N+1 query patterns” as review findings. citeturn27view0

**Documentation and spec drift (README/ADR/OpenAPI mismatch, misleading comments, incomplete migrations docs)**  
What it is: externally visible (or internal) documentation that falls out of sync with actual behavior, breaking downstream user expectations and future maintenance. citeturn27view0turn6search3turn9view0  
Why it escapes: doc correctness is rarely tested; linters might enforce formatting but not semantic alignment; ticket verification often ignores docs unless explicitly required. Mäntylä & Lassenius include “documentation defects” in the evolvability class, again not detectable via execution-based QA. citeturn9view0turn9view1  
Concrete evidence/example: Ellipsis explicitly claims to detect “documentation drift,” and Graphite includes “inconsistencies between code behavior and documentation” as a logic-bug class. citeturn6search3turn27view0

**Accidentally committed or “shouldn’t ship” artifacts (debug logs, dev config, commented code, temporary workarounds)**  
What it is: dead code, temporary debug statements, dev-only config, fixtures, commented-out blocks—anything that is harmless to tests but harmful to production clarity and risk posture. citeturn27view0turn32view0  
Why it escapes: tests generally don’t assert “absence of debug logs” or “no dev config shipped.” This is a classic review-time hygiene catch. citeturn27view0  
Concrete evidence/example: Graphite lists “accidentally committed code” (debug statements, dev configurations, temporary workarounds) as a specific review category. citeturn27view0

**System-level consistency and “one PR, one story” integrity (scope creep, conflicting changes, inconsistent naming/UX semantics)**  
What it is: the epic PR as a whole violates system invariants: inconsistent naming conventions across newly added components, mixed error-handling styles across tickets, diverging UX semantics, or extra surface area beyond the tickets’ stated intent. citeturn6search0turn11view0  
Why it escapes: each ticket can be locally “in scope,” yet the aggregate diff adds up to drift. Bacchelli & Bird’s study highlights that review is heavily about *understanding the change* and its context; tooling often fails to meet these understanding needs, which is exactly where PR-level review must focus. citeturn11view0  
Concrete evidence/example: Google’s “standard of code review” frames review as ensuring the change improves overall code health—even if not perfect—reinforcing that review is a system health gate, not just “tests passed.” citeturn6search4

## Rejection feedback pattern comparison

PR rejection in a no-human pipeline is only useful if it becomes structured work that reliably re-enters execution. The practical bar is: deterministic, parseable, de-duplicated, and scoped small enough that sub-agents can fix without creating new chaos.

**Pattern A: Create new corrective tickets (preferred for closed-loop autonomy)**  
Mechanism: PR review agent emits one or more new `.tickets/*.md` files (YAML frontmatter + markdown body) describing each corrective action, with explicit verification commands and acceptance criteria.  
Tradeoffs: strongest auditability and cleanest closed loop; enables parallelism (multiple fix tickets can run concurrently). Primary risk: ticket explosion and duplicated issues unless you dedupe by fingerprint (file path + rule + snippet hash) and enforce severity thresholds. citeturn32view0turn3view0  
Evidence anchor: Sourcery supports turning a review comment into a GitHub issue on demand (`@sourcery-ai issue`), which is essentially “convert review feedback into structured backlog.” citeturn23view0

**Pattern B: PR comments that map to actionable work items (good UX, weak structure unless enforced)**  
Mechanism: PR review agent posts a single summary comment plus inline comments. An orchestrator parses comments and converts them into tickets.  
Tradeoffs: minimal friction; leverages native PR UI. But comments are semi-structured, can be noisy, and parsing is brittle unless you enforce a strict schema (e.g., a fenced JSON payload). The failure mode is “the bot wrote prose; the orchestrator guessed wrong.” citeturn11view0turn30view1  
Evidence anchor: Graphite and Copilot both deliver feedback as PR line comments with “what/why/how to fix,” and allow suggested changes to be applied/committed—strong for interaction, but not inherently a backlog loop. citeturn27view0turn30view1

**Pattern C: Reopen the epic as a new execution cycle (fast, but easy to create infinite loops)**  
Mechanism: PR review agent rejects; orchestrator re-runs agents against the same epic branch with a “fix to satisfy these findings” instruction set.  
Tradeoffs: fast to implement and avoids ticket proliferation. But it is not stable for long-running systems because you lose a durable, queryable defect backlog; you also risk oscillation (agents “fix” and “re-break”) unless you pin acceptance criteria and verification. This is precisely why staged/filtered review systems evolved: unfiltered feedback erodes trust and creates churn. citeturn3view0turn24view0

**Pattern D: Direct orchestrator notification with a retry instruction (lowest overhead, lowest traceability)**  
Mechanism: PR review agent sets a failing status check and attaches a machine-readable payload; orchestrator consumes it and schedules fix work without creating tickets unless configured.  
Tradeoffs: simplest wiring; best for a lightweight MVP. But without ticket materialization you lose historical learning (“what keeps failing?”) and you can’t amortize improvements into process (new rules, new tests). Modern systems tend to add dashboards and feedback loops once scale arrives. citeturn27view0turn3view0

**Recommendation for a lightweight first implementation**  
Start with a hybrid: **(1) failing required status check + (2) one machine-readable rejection payload + (3) orchestrator converts payload into new `.tickets/` files automatically.** This gives you immediate merge blocking (deterministic) and structured work (ticket-native) without requiring the PR review agent to push commits. citeturn34search2turn34search5turn30view1  
Minimal implementation details:
- A required status check named `pr-review-agent` is enforced via branch protection/rulesets so merges are blocked unless it succeeds. citeturn34search2turn34search5turn34search0  
- The PR review agent posts exactly one PR comment containing a strict JSON object (schema below in “Recommended protocol”).  
- The orchestrator reads that JSON, generates tickets, and triggers agents.  
This mirrors what Sourcery does conceptually (turn comments into issues) but makes it deterministic and repo-native for agent execution. citeturn23view0turn32view0

## Existing tool analysis

This section focuses on what today’s AI PR review products do well for feedback loops—and what breaks—specifically through the lens of “no human reviewer, PR is the terminal gate.”

image_group{"layout":"carousel","aspect_ratio":"16:9","query":["CodeRabbit AI code review GitHub app screenshot","Graphite AI Reviews Graphite Agent screenshot","GitHub Copilot code review pull request screenshot","Sourcery AI code review pull request screenshot","Ellipsis.dev AI code review screenshot"],"num_per_query":1}

**entity["company","CodeRabbit","ai code review tool"]**  
What it does well:
- Strong “context engineering” story: code graph (definitions/references), commit-history co-change signals, semantic index of functions/tests/prior PRs, and explicit “verification scripts” for evidence-backed comments. This directly targets PR-level cross-file/cross-history failures. citeturn25view0turn29view2  
- Multi-repo analysis as a first-class feature, with explicit examples (API contracts, microservices, shared libraries, schemas). This aligns closely with cross-ticket consistency failures. citeturn29view1  
- “Learnings” as incremental memory: reviewers can correct preferences and CodeRabbit persists them for future reviews (an operational approach to reducing recurring false positives). citeturn29view0  
Failure modes to plan around:
- Like all LLM systems, quality depends on relevance filtering; the product itself emphasizes avoiding diff-only review. Your custom agent must make context retrieval explicit and bounded or it will drown in its own “helpfulness.” citeturn25view0turn12view0

**entity["company","Graphite","ai code review platform"]**  
What it does well:
- Clear, review-friendly comment structure: problem + why it matters + concrete fix; supports committing suggestions “like teammates.” citeturn27view0  
- Explicit taxonomy of bug types that “slip through manual code review and testing,” including edge cases (race conditions), security classes, performance (N+1), and “accidentally committed code.” This is essentially a built-in defect taxonomy your PR agent can borrow. citeturn27view0  
Failure modes:
- Context limits are real: Graphite notes PRs may not be analyzed if they exceed a size threshold (example: >200,000 characters). A terminal gate must either chunk/stage or reject on “review capacity exceeded.” citeturn27view0  
- These systems are designed assuming humans choose what to do; your pipeline cannot. You need deterministic “reject” semantics, not just comments. citeturn26view0turn34search2

**entity["company","GitHub Copilot","ai coding assistant"]** (code review feature)  
What it does well:
- Native integration: can be added as a PR reviewer; supports feedback reactions; supports automatic review triggers; supports re-review requests. citeturn30view1turn30view2  
- Repository instructions exist, but with hard constraints (e.g., only first 4,000 characters are read) and acknowledged non-determinism. citeturn30view1turn28view0  
Failure modes (and why they matter more in a no-human loop):
- Copilot “always leaves a Comment review,” not “Approve/Request changes,” and *does not block merging*; it is explicitly not a quality gate by itself. citeturn30view1  
- Re-review is not automatic by default; it may repeat comments even after dismissal/downvote. That is toxic in an autonomous loop because it creates infinite “fix/review/repeat” churn unless you add dedupe fingerprints and state. citeturn30view1turn30view0  
- The product itself advises supplementing with human review—exactly what your pipeline removes—so you must compensate with stronger staged checks and structured remediation. citeturn30view2turn3view0

**entity["company","Sourcery","ai code review tool"]**  
What it does well:
- Multi-reviewer architecture: several specialized reviewers (quality, security, complexity, docs, testing) plus a validation process to reduce false positives. This aligns with staged review best practice. citeturn6search2  
- Crucially for loop closure: explicit commands to trigger re-review and to convert a review comment into a GitHub issue (`@sourcery-ai issue`). That is a concrete pattern of “rejection → structured follow-up work.” citeturn23view0  
Failure modes:
- As with other comment-first tools, the “issue creation” is user-triggered, not automatic; your agent must automatically decide when to create new tickets/issues to avoid blocked merges in a no-human setup. citeturn23view0turn34search2

**entity["company","Ellipsis","ai code review tool"]**  
What it does well:
- A very explicit architecture for handling the #1 practical failure mode: false positives. Ellipsis runs multiple comment generators in parallel and then a multi-stage filtering pipeline (including hallucination filtering) and includes evidence links. citeturn24view0  
- It indexes pull requests and uses retrieval to find prior examples (“when did we do X”), showing a concrete implementation of historical context retrieval for review quality. citeturn24view0  
Failure modes:
- Their own writeup emphasizes accuracy over latency and highlights that large-context prompts degrade model performance; your PR review agent should therefore *not* be a single giant prompt over the entire epic diff. citeturn24view0turn12view0

**entity["company","Aviator","developer workflow tools"]** (FlexReview/MergeQueue context)  
Aviator is not primarily an LLM content reviewer in the same way as the above tools; it’s workflow infrastructure that uses PR history and heuristics to assign reviewers and validate approvals (FlexReview) and manage merges (MergeQueue). This is directly relevant to autonomous pipelines because it shows how “review outcomes” become enforceable status checks and policies. citeturn5search0turn5search1turn5search9

Key practitioner lesson across tools: **noise kills loop closure**. HubSpot’s writeup is unusually explicit: early automated reviews were “overly effusive…verbose…nitpicky,” and they introduced a second “Judge Agent” gating comments on succinctness/accuracy/actionability—described as “arguably the single most important factor” in effectiveness. citeturn3view0 This matches Ellipsis’ and Sourcery’s multi-stage filtering/validation approach. citeturn24view0turn6search2

## Recommended protocol

This is a concrete specification for a PR review agent that acts as the terminal gate before merging an epic branch to main, assuming: sub-agents already ran ticket commands; orchestrator reran them and checked scope; PR agent must catch cross-ticket/system-level issues and push rejections back into `.tickets/` as structured work.

### Review model

**Staged, not single pass (evidence-backed)**  
Use a staged pipeline because both research and real-world implementations show that unfiltered LLM feedback is noisy and repetitive, and that adding a judge/filter stage materially improves usefulness. citeturn3view0turn24view0turn6search2  
Also stage because deterministic tooling (SAST/linters/coverage) is cheaper and can fail fast, while LLM reasoning should be reserved for the high-value “understanding/design/consistency” checks that tools struggle with. Static analysis warnings can have substantial false positives (Google research cites ≥30% in some tools) and require triage; recent industrial evidence shows LLM+static hybrids can reduce false positives dramatically while staying cost-effective—supporting “tools first, LLM adjudication second.” citeturn17view0turn18view0

### Checklist in priority order

The PR review agent evaluates the epic PR at three tiers:

**Tier 0: Merge-blocking invariants (deterministic)**
1. **All required CI checks pass on the latest SHA**; if not, reject without further analysis. citeturn34search0turn34search2  
2. **Epic-level diff hygiene**: no secrets added (regex/supply-chain scanners), no debug-only artifacts, no accidental dev config shipped. (Tools exist; the key is enforcing at PR scope as a gate.) citeturn27view0turn6search17  
3. **Policy compliance**: forbidden dependency/license categories, banned APIs, disallowed file paths. (Repo-specific policy file is authoritative.) citeturn6search0turn34search5

**Tier 1: Cross-ticket/system integration checks (agentic + retrieval)**
4. **Contract surface audit**: detect changes to public interfaces (routes, DTOs, schemas, events) and confirm corresponding updates exist (migrations, clients, docs). Prefer retrieval over guesswork: search for dependents and co-changing files. citeturn29view1turn25view0turn24view0  
5. **Cross-ticket consistency scan**: naming/error-handling/logging patterns; config conventions; shared helper semantics; feature flag behavior. This is explicitly “design and consistency” review, per Google guidance. citeturn6search0turn6search4  
6. **Test adequacy gate**: for each new/changed behavior class, require at least one of: unit test, integration/contract test, or explicit waiver in ticket history. (“Tests passed” is insufficient; you need *tests exist for what changed*.) citeturn6search0turn27view0turn1search28

**Tier 2: Deep reasoning pass (LLM, filtered by a judge)**
7. **Security reasoning review**: authZ invariants, injection surfaces, crypto usage, sensitive logging. Anchor to known secure coding categories (OWASP) and flag deviations. citeturn6search17turn33view0  
8. **Performance risk review**: identify N+1 and hot-path regressions; require justification or remediation ticket. citeturn27view0  
9. **Evolvability review**: complexity, duplication, documentation accuracy, readability—because these are a majority share of review-discovered defects and escape execution-based QA. citeturn9view0turn32view0turn6search0  

All Tier 2 findings must pass a **judge filter**: *actionable, precise, and non-duplicative*. This is directly supported by HubSpot’s evaluator-optimizer (“Judge Agent”) approach and by Ellipsis’ multistage filtering pipeline design. citeturn3view0turn24view0

### Historical context retrieval policy

Historical retrieval is valuable, but only if bounded and relevance-filtered.

**Evidence**: CodeRabbit and Ellipsis both index prior changes/PRs to ground reviews in “how this repo solved similar problems before,” and Copilot offers memory/instructions for repository context. citeturn25view0turn24view0turn30view2 Research on retrieval-augmented review generation shows measurable quality gains on benchmarks (e.g., improved BLEU/exact match and better handling of low-frequency tokens), but also warns that adding more retrieved exemplars can *hurt* due to redundancy/conflicting cues. citeturn14view0turn12view0  
**Protocol rule**: retrieve **top-1 to top-3** exemplars per finding cluster, dedupe by similarity, and require an explicit “evidence link set” (paths + symbol references + commit/PR references). If retrieval confidence is low, downgrade severity and create a “needs human-quality signal” ticket rather than blocking merge on a guess. citeturn12view0turn24view0

### Rejection signaling mechanism (exact spec)

**Blocking signal**: a required status check named `pr-review-agent` with conclusion `failure` on rejection; this is enforced by branch protection/rulesets so merging is impossible if the check fails. citeturn34search2turn34search5turn34search0

**Human-readable signal**: one PR comment titled `PR REVIEW AGENT VERDICT` containing:
- a short prose summary (≤150 words)
- a fenced JSON payload that is the *source of truth* for the orchestrator

**Machine-readable payload schema (minimum viable)**

```json
{
  "verdict": "REJECT",
  "epic_branch": "epic/<name>",
  "base_branch": "main",
  "head_sha": "<sha>",
  "findings": [
    {
      "id": "F-001",
      "severity": "BLOCKER",
      "category": "CONTRACT_MISMATCH",
      "title": "Backend endpoint changed without client update",
      "evidence": {
        "files": ["path/a.ts", "path/b.ts"],
        "symbols": ["getUser", "UserDTO"],
        "related_history": ["pr:#1234", "commit:abcd123"]
      },
      "fix_strategy": "NEW_TICKET",
      "ticket_blueprint": {
        "slug": "fix-contract-userdto",
        "frontmatter": {
          "type": "bugfix",
          "priority": "p0",
          "owner": "agent",
          "verification": ["pnpm test", "pnpm lint"]
        },
        "acceptance_criteria": [
          "All clients compile and tests pass",
          "Docs updated (OpenAPI/README) to match behavior"
        ]
      }
    }
  ],
  "dedupe_fingerprint": "sha256:<...>",
  "recheck_instructions": "Re-run pr-review-agent after all generated tickets are marked done."
}
```

### Rejection re-entry flow (step-by-step)

1. PR review agent runs staged review. If Tier 0 fails → sets `pr-review-agent=failure` with a single finding (“CI not green”) and stops. citeturn34search0turn34search2  
2. If Tier 1/2 finds BLOCKERs → agent emits the PR comment + JSON payload and sets `pr-review-agent=failure`.  
3. Orchestrator watches for `pr-review-agent=failure`, fetches the JSON payload, and **materializes tickets** into `.tickets/`:
   - One ticket per BLOCKER finding by default (configurable batching by category).
   - Each ticket includes the exact verification commands to run and explicit acceptance criteria.  
4. Orchestrator opens a new execution cycle:
   - checks out isolated worktrees per new ticket
   - dispatches sub-agents to implement fixes
   - runs ticket verification commands
   - merges passing work back into the epic branch
   - updates the PR head branch.  
5. Orchestrator then triggers a PR re-review:
   - if using GitHub-native reviewers, request re-review explicitly (many systems do not auto re-review on push unless configured). citeturn30view1turn30view2  
   - regardless, rerun `pr-review-agent` as a required status check. citeturn34search0turn34search5  
6. Dedupe and loop prevention:
   - The orchestrator stores `dedupe_fingerprint` values; if the same fingerprint fails N times, it escalates by generating a single “meta-ticket” to add a missing invariant test/rule (e.g., contract test, migration check) instead of endlessly re-fixing symptoms. This is consistent with research showing many review-detected concerns could be automated, but still reach review due to tool gaps/adoption barriers. citeturn32view0turn17view0  
7. Success condition:
   - `pr-review-agent=success` and no BLOCKER/MAJOR findings remain; only MINOR/INFO may be allowed depending on policy. The PR can now merge because required checks are green. citeturn34search2turn34search0

**Explicit design stance (reasoned extrapolation)**: do *not* rely on PR “Request changes” reviews as the only gate, because products differ in whether bot reviews block merges (Copilot explicitly does not), and because status checks are the most deterministic merge barrier under branch protection. citeturn30view1turn34search2