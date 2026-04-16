# Adversarial Review: Ticket Workflow Improvement Proposal

## Your role

You are a senior infrastructure engineer and skeptical reviewer. You have been handed an improvement proposal for a ticket workflow system and asked to attack it. Your job is not to validate; your job is to find what breaks, what's unjustified, what's internally inconsistent, and what's worse than the status quo. The author has already convinced themselves. Your value comes from the opposite stance.

Assume the author is technically competent but has motivated reasoning — they read the research, synthesized it, and are now emotionally invested in their own conclusions. Find the conclusions that don't survive contact with the evidence.

---

## Input set (read all before writing anything)

Read these files in this order. Do not skim. The review's value is proportional to how much of the evidence you actually checked against the claims.

**The proposal you are reviewing:**
- `docs/improvement_proposal_apr_2026.md` — the three-part plan (Immediate fixes, Research-backed improvements, Implementation sequence)

**The evidence the proposal claims to rest on:**
- `docs/adversarial_review_apr_2026.md` — the prior adversarial review identifying active defects and research directions
- `docs/research/r1-pr-review-agent.md`
- `docs/research/r2-orchestrator-review-protocol.md`
- `docs/research/r3-decomposition-calibration.md`
- `docs/research/r4-archive-knowledge-retrieval.md`
- `docs/research/r5-worktree-failure-recovery.md`

**The current system the proposal modifies:**
- `.claude/skills/ticket-workflow/SKILL.md`
- `.claude/skills/ticket-workflow/references/templates.md`
- `AGENTS.md`

---

## Stated constraints the proposal must honor

These came from the synthesis prompt and are binding on the proposal. Flag any violation.

1. **No persistent infrastructure.** Every improvement must work without an always-on server, a persistent database, or services outside a standard git repository with a Python or Node runtime. On-demand computation is allowed; a permanent background process is not.
2. **Specific file references required.** Every "Change" section must name files that exist in the current system or name new files with explicit paths. No hypothetical files.
3. **No further research.** No "investigate X" or "research Y" as an improvement.
4. **Specificity over completeness.** Fewer precise proposals > many vague ones.

---

## What to attack

Walk each of these axes independently. Do not merge them into a single "looks good/bad" verdict.

### 1. Evidence alignment — does each proposal match what the research actually says?

For each of Part 2 items A–E, open the cited research document and verify:
- Is the "Research basis" claim accurate? If R2 is cited, is the artifact (12 gates, 4 escalation actions, prompt template) actually in R2 and in the form claimed? Are the numbers (12, 4, 6, 8-factor, 25 rounds, 3 files, 5 files, 120–200 words) *in the research*, or extrapolated?
- Does the proposal drop important nuances? Example: R3 distinguishes "Tier A" (capable-but-limited) vs "Tier B" (frontier) models — does the proposal honor both, or collapse them?
- Does the proposal add claims the research does not support? Example: "N=3" for dedupe trigger — R1 says "N times" without binding. Is N=3 justified?
- Does the proposal misread a research recommendation? Look for cases where the proposal appears to implement X but the research actually recommends Y.

### 2. Success criteria — do they actually verify the intended outcome?

For each proposal, examine the "Success criteria" bullets. Each should be independently verifiable by a command or observable state, *and* the command should actually confirm the change achieved its purpose, not just that text was added.

- Example attack vector: "`grep -c 'complexity ≥8' SKILL.md` returns ≥1" verifies a string exists. It does not verify that agents will actually apply the threshold. Is that gap acceptable, or does it render the success criterion hollow?
- Example attack vector: proposal A requires `.prompts/orchestration/_template.md` exists with "six placeholder tokens present" — but only five are listed in the bullet. Count and flag mismatches.
- For each proposal, name at least one way the success criteria could pass while the proposal's actual goal is unmet.

### 3. Internal consistency — do proposals conflict with each other or the existing system?

Look for collisions:
- Does proposal A create a file or section that proposal B assumes does not exist, or vice versa?
- Does a Part 1 immediate fix contradict a Part 2 structural change?
- Does a proposal add a step to the execution protocol that would conflict with existing protocol steps (e.g., Step 7 sub-step for Outcome vs Step 7 dedicated commit rule)?
- Does the implementation sequence in Part 3 require work that the earlier items haven't delivered?
- Does any proposal re-introduce something the adversarial review explicitly flagged as a defect (e.g., referencing a file that may not exist in a given workspace, like `docs/INDEX.md`)?

### 4. Scope, cost, and feasibility

- Is any proposal underspecified enough that an agent implementing it would improvise in the wrong direction? Which "Change" paragraphs are vague in ways that matter?
- Is any proposal actually larger than a single ticket should be? R1+R2+R3 each propose multiple new files and SKILL.md sections — can they be implemented by a single execution agent under the 200-line ticket constraint and the "decompose if 4+ files" rule (which this proposal plans to replace but which is still operative today)?
- What's the reading burden on agents? Count how many new reference files the proposal adds and how much SKILL.md grows. Does the skill still fit the stated "workspace-agnostic portable" goal?
- Does proposal D's shell script (`archive-search.sh`) or the embedding lane (`--semantic`) introduce dependencies (sentence-transformers, Python env) that violate "no persistent infrastructure" in spirit if not in letter? What's the install burden in a cold workspace?
- Proposal E references a `.git/config.lock` serialization rule — but the orchestrator is itself an LLM agent running shell commands. How does "serialize `git worktree add` per repo" get enforced when the serializer is the agent that's supposed to obey the rule?

### 5. Missing items, wrong priorities, and status-quo bias

- Did the proposal drop any defect from the adversarial review that should have been included? (Check the full list of active defects #1–#7 and the Immediate Fixes list.)
- Is any Part 1 priority defensible? For example, is "Remove `docs/INDEX.md`" really **high priority** — or is it a one-line edit that is simply easy, not important?
- Is the ordering in the Implementation Sequence (TASK template → orchestrator → PR review agent) actually optimal, or does it privilege "keystone unblocker" framing over a different cut (e.g., decomposition calibration first because it governs every subsequent ticket's size)?
- Are any critical gaps invisible because they weren't in the research reports? The research directions were *selected* by the prior review — what important gaps did the prior review miss and the proposal therefore perpetuates?

### 6. Second-order effects and failure modes

For each research-backed improvement, name at least one realistic failure mode the proposal does not address:
- **A (Orchestrator protocol):** What happens when the orchestrator itself drifts from the protocol — who reviews the orchestrator?
- **B (PR review agent):** What stops the rejection → materialize-tickets → re-execute loop from oscillating if the PR review agent's judgment is inconsistent across runs? Dedupe by fingerprint helps for identical findings; what about *near-identical* findings?
- **C (Complexity score):** The score is agent-authored. What's the calibration mechanism? Every agent is incentivized to score low (avoid the cost of decomposition) or high (avoid the risk of failure). Which way does it drift?
- **D (Outcome section):** If every ticket gets a 200-word Outcome, the archive grows 200 words × N tickets × M epics. At what scale does "ripgrep is fast enough" break down? At what scale does the on-demand embedding build become intolerable? Is that horizon named?
- **E (Worktree runbook):** The runbook tells agents how to recover. How does an agent know *when* to invoke it — is there a trigger, or does the agent have to notice on its own? What's the failure mode where an agent doesn't realize it's in a recovery state?

### 7. Violation of the constraints listed above

Explicitly check each constraint against each proposal and flag any violations. Be literal: "no persistent infrastructure" excludes a persisted vector index on disk, but allows on-demand indices. "Specific file references required" means every `references/…` file named in the proposal must either exist or have an explicit new path — check each one.

---

## Output format

Produce one document with these sections, in this order. Do not invert or skip.

### Section 1: Summary verdict

2–4 sentences. State overall whether the proposal is ready to become tickets, ready with revisions, or requires substantial rework. Name the single most load-bearing weakness.

### Section 2: Critical issues (blockers)

Issues that would make the proposal harmful or unimplementable if shipped as-is. For each:
- **Title** — ≤12 words
- **Proposal affected** — e.g., "Part 1 #2" or "Part 2 A"
- **Problem** — 2–4 sentences, evidence-grounded (cite the research file + section or the current system file)
- **Suggested fix** — 1–2 sentences, concrete; no hand-waves

### Section 3: Serious issues (should-fix)

Issues that weaken the proposal but don't block it. Same format as Section 2.

### Section 4: Minor issues and nits

Specificity problems, typos in success criteria, small inconsistencies. Bulleted list; one line each.

### Section 5: What the proposal got right

Brief. Counter-signal to confirm you engaged the material honestly rather than attacking for the sake of attacking. 3–6 bullets.

### Section 6: Re-prioritization recommendation

If you believe the Part 3 implementation sequence should change, state the new sequence (exactly three items, in order) and justify in ≤100 words why it's better than the proposed sequence.

---

## Posture rules

1. **Cite, don't assert.** Every critique must point to either a specific line/section in the proposal, a specific claim in the cited research, or observable state of the current system. If you can't cite, you can't critique.

2. **Quantify where possible.** "The archive may get big" is weak. "At 200 words × 10 tickets × 100 epics = 200K words, the embedding build exceeds the latency budget R4 claims" is useful.

3. **No new research.** You are attacking the synthesis, not redoing it. Do not propose reading additional papers, running experiments, or surveying more tools. Work with the evidence in the input set.

4. **Don't rewrite the proposal.** Point out what's wrong. Suggest fixes only as short gestures. A full rewrite is out of scope and wastes tokens.

5. **Resist fake rigor.** Do not invent false-precision numbers (e.g., "this has a 40% chance of X") when the evidence does not support them. "The evidence doesn't speak to this; the proposal assumes Y without justification" is a stronger attack than fabricated quantification.

6. **Spend the thinking budget.** You are Extra High Thinking. Trace the research → proposal claim chain carefully for at least A, B, and C. Reviewing five proposals shallowly is worse than reviewing three proposals with depth plus an honest "I didn't rigorously check D and E" note.

7. **Name your uncertainty.** If you're not sure whether something is a defect or a judgment call, say so. Flagging "this might be fine, but I can't verify from the inputs" is better than silence or overclaiming.

---

## Anti-patterns to avoid in your output

- Praising the proposal's "ambition" or "thoroughness" instead of engaging specifics.
- Restating what the proposal says before critiquing it. Assume the reader has read the proposal; go straight to the critique.
- Generic commentary about "agentic systems" or "code review best practices" that doesn't tie back to this specific proposal.
- Vague imperatives like "should consider" or "might want to think about." Say what's wrong and what a concrete fix looks like.
- Producing a long document with low information density. Terse is better than verbose.

---

## Delivery

Write the output in markdown, suitable for saving as `docs/adversarial_review_of_proposal_apr_2026.md`. Do not include any preamble or meta-commentary ("Here is my review…"). Start with the Section 1 heading.
