You are a software engineer executing two structured tickets to create a Claude Code skill and refactor an existing file. You will implement both tickets in dependency order — FEAT-001 first, then REFAC-001 — and verify each before moving on.

<documents>
<document index="1">
<source>.tickets/FEAT-001_ticket-workflow-skill.md</source>
<document_content>
---
id: FEAT-001
title: "Create ticket-workflow SKILL.md for the .tickets/ system"
type: feature
status: to-do
priority: high
created: 2026-03-26
updated: 2026-03-26
parent:
dependencies: []
tags: [skills, tickets, agent-workflow]
agent_created: false
complexity: 4
---

# Create ticket-workflow SKILL.md for the .tickets/ system

## Context
The full ticket decomposition and execution protocol currently lives inline
in `AGENTS.md` (lines 29-67). This makes AGENTS.md long and mixes high-level
project guidance with detailed procedural instructions. A dedicated SKILL.md
moves the detailed protocol into a skill that agents load on demand, keeping
AGENTS.md lean and scannable.

The skill should be usable by Claude Code natively (via `.claude/skills/`)
and portable to Codex (via `.agents/skills/` symlink or copy). Cursor agents
don't consume SKILL.md directly but can read the file through file operations
when pointed to it by a Cursor rule.

## Requirements
- [ ] Create `.claude/skills/ticket-workflow/SKILL.md` with proper frontmatter (`description` field that triggers when the user asks to work on a ticket, create a ticket, or manage tickets)
- [ ] The skill must contain the complete ticket execution protocol currently in AGENTS.md lines 29-67 ("Working on a ticket" through "Creating sub-tickets"), restructured as clear agent instructions
- [ ] The skill must include the ticket quality rules currently in AGENTS.md lines 59-67
- [ ] The skill must document the naming convention (`TYPE-NNN_kebab-slug.md`) and all valid type prefixes (`FEAT`, `BUG`, `REFAC`, `CHORE`, `TASK`, `EPIC`)
- [ ] The skill must document the status lifecycle (`to-do` → `in-progress` → `review` → `done`, plus `blocked`)

## File path hints
- `.claude/skills/ticket-workflow/SKILL.md` — create (primary deliverable)
- `AGENTS.md` — read only in this ticket (do NOT modify; that is REFAC-001)
- `.tickets/_templates/` — read only (reference these templates in the skill)

## Constraints
- Do NOT modify `AGENTS.md` — that is handled by a separate ticket (REFAC-001)
- Do NOT modify or reorganize any existing files
- Do NOT create Cursor rules or Codex-specific files — keep scope to the SKILL.md only
- The skill file must be self-contained: an agent reading only this file should have everything it needs to execute the ticket protocol
- Do NOT duplicate the template contents into the skill — reference `.tickets/_templates/` and instruct the agent to read the appropriate template

## Acceptance criteria
- [ ] `.claude/skills/ticket-workflow/SKILL.md` exists with valid frontmatter including a `description` field
- [ ] An agent given "work on ticket FEAT-001" can follow the skill to read, assess, decompose, execute, and verify the ticket without needing to consult AGENTS.md for procedural details
- [ ] The skill references `.tickets/_templates/` for ticket creation rather than inlining template content
- [ ] The skill is under 200 lines

## Verification
```bash
head -5 .claude/skills/ticket-workflow/SKILL.md | grep -q "^---"
test $(wc -l < .claude/skills/ticket-workflow/SKILL.md) -le 200
grep -q "dependencies" .claude/skills/ticket-workflow/SKILL.md
grep -q "decompose" .claude/skills/ticket-workflow/SKILL.md
grep -q "Verification" .claude/skills/ticket-workflow/SKILL.md
grep -q "_templates" .claude/skills/ticket-workflow/SKILL.md
echo "All checks passed"
```

## Notes
- The skill description field should read something like: "Use when working on structured tickets from the .tickets/ directory. Handles ticket reading, decomposition into sub-tickets, status updates, and verification."
- Keep the skill focused on *procedure* (how to work on tickets), not *policy* (coding standards) — AGENTS.md retains ownership of general coding standards
</document_content>
</document>
<document index="2">
<source>.tickets/REFAC-001_slim-agents-md.md</source>
<document_content>
---
id: REFAC-001
title: "Reduce AGENTS.md ticket protocol to summary and skill pointer"
type: refactor
status: to-do
priority: high
created: 2026-03-26
updated: 2026-03-26
parent:
dependencies: [FEAT-001]
tags: [agents-md, skills, tickets]
agent_created: false
complexity: 2
---

# Reduce AGENTS.md ticket protocol to summary and skill pointer

## Motivation
AGENTS.md currently contains the full ticket decomposition protocol inline
(~40 lines of detailed procedural instructions). Now that FEAT-001 has moved
this protocol into `.claude/skills/ticket-workflow/SKILL.md`, AGENTS.md should
be slimmed down to a brief summary and a pointer to the skill. This keeps
AGENTS.md scannable and under ~40 lines total, reducing context overhead for
every agent session since AGENTS.md is always loaded at startup.

## Scope
- `AGENTS.md` — the "Ticket System (.tickets/)" section (currently lines 7-67)

## Target structure

The "Ticket System (.tickets/)" section in AGENTS.md should be reduced to:

1. **One-line description** of the ticket system (structured markdown tickets in `.tickets/`)
2. **Ticket types table** (keep the existing 6-row table — it's a quick reference agents need at startup)
3. **Status lifecycle** one-liner (keep as-is)
4. **Pointer to the skill**: a short paragraph directing agents to read `.claude/skills/ticket-workflow/SKILL.md` for the full ticket execution protocol (reading, decomposition, sub-ticket creation, verification)
5. **Templates pointer**: one line noting templates live in `.tickets/_templates/`

The following subsections should be **removed entirely** from AGENTS.md (their content now lives in the skill):
- "Working on a ticket" (steps 1-7)
- "Creating sub-tickets" (the sub-ticket table example)
- "Quality rules" (the bullet list)

The "General Coding Standards" section at the bottom of AGENTS.md must remain unchanged.

## Acceptance criteria
- [ ] AGENTS.md "Ticket System" section contains the types table, status lifecycle, and a pointer to the skill
- [ ] AGENTS.md "Ticket System" section does NOT contain the 7-step execution protocol, sub-ticket creation details, or quality rules
- [ ] AGENTS.md "General Coding Standards" section is completely unchanged
- [ ] AGENTS.md total length is under 45 lines

## Constraints
- Do NOT modify the "General Coding Standards" section
- Do NOT modify any files other than `AGENTS.md`
- Do NOT change the heading structure (keep `## Ticket System (.tickets/)` as-is)
- Do NOT add new content — this is a pure reduction

## Verification
```bash
test $(wc -l < AGENTS.md) -le 45
grep -q "FEAT.*feature" AGENTS.md
grep -q "to-do.*in-progress.*review.*done" AGENTS.md
grep -q "skills/ticket-workflow" AGENTS.md
! grep -q "Assess complexity" AGENTS.md
! grep -q "Mark parent done" AGENTS.md
grep -q "General Coding Standards" AGENTS.md
echo "All checks passed"
```

## Notes
- This ticket depends on FEAT-001 — the skill must exist before AGENTS.md can point to it
- The goal is progressive disclosure: AGENTS.md gives agents enough context to know the ticket system exists and what types/statuses are valid; the skill provides the detailed protocol when they actually need to execute
</document_content>
</document>
</documents>

<instructions>
Execute these two tickets sequentially. FEAT-001 has no dependencies and must be completed first. REFAC-001 depends on FEAT-001.

For each ticket, follow this exact sequence:

1. Read the ticket fully — it is provided above, so you already have it in context.
2. Read any files referenced in the ticket's "File path hints" section before writing code.
3. Implement the changes described in "Requirements" (FEAT-001) or "Target structure" (REFAC-001).
4. Run every command in the ticket's "Verification" section and confirm they all pass.
5. Update the ticket file's frontmatter: set `status: done` and `updated:` to today's date.

When creating the SKILL.md for FEAT-001:
- Start with YAML frontmatter containing a `description` field. The description should trigger when the user asks to work on a ticket, create a ticket, or manage tickets in the `.tickets/` directory.
- The body should contain the complete ticket execution protocol (read → check dependencies → assess complexity → decompose → execute → verify → mark done), restructured as clear procedural instructions an agent can follow step-by-step.
- Include the naming convention, type prefixes, and status lifecycle.
- Include the quality rules for ticket authoring.
- Reference `.tickets/_templates/` for ticket creation — do not inline template content.
- The file must be under 200 lines and fully self-contained.

When refactoring AGENTS.md for REFAC-001:
- Keep: the heading, one-line description, ticket types table, status lifecycle one-liner, and the "General Coding Standards" section verbatim.
- Add: a short paragraph pointing to `.claude/skills/ticket-workflow/SKILL.md` for the full execution protocol, and one line noting templates live in `.tickets/_templates/`.
- Remove: "Working on a ticket" steps, "Creating sub-tickets" subsection, and "Quality rules" subsection.
- The result must be under 45 lines total.

After both tickets pass verification, confirm completion with a brief summary of what was created and changed.
</instructions>

<guardrails>
- Implement changes directly. Do not suggest changes or ask for confirmation — act.
- Do not create files beyond what the tickets specify. No helper scripts, no temporary files, no extra documentation.
- Do not modify files the tickets say to leave alone. FEAT-001 says do not touch AGENTS.md. REFAC-001 says do not touch anything except AGENTS.md.
- If a verification command fails, fix the issue and re-run. Do not skip verification.
- Do not add features, abstractions, or improvements beyond what the tickets specify.
- Choose an approach and commit to it. Avoid revisiting decisions unless verification fails.
</guardrails>
