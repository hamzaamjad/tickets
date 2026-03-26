---
id: REFAC-001
title: "Reduce AGENTS.md ticket protocol to summary and skill pointer"
type: refactor
status: done
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
# AGENTS.md is under 45 lines
test $(wc -l < AGENTS.md) -le 45
# Types table still present
grep -q "FEAT.*feature" AGENTS.md
# Status lifecycle still present
grep -q "to-do.*in-progress.*review.*done" AGENTS.md
# Skill pointer present
grep -q "skills/ticket-workflow" AGENTS.md
# Detailed protocol removed
! grep -q "Assess complexity" AGENTS.md
! grep -q "Mark parent done" AGENTS.md
# General Coding Standards untouched
grep -q "General Coding Standards" AGENTS.md
echo "All checks passed"
```

## Notes
- This ticket depends on FEAT-001 — the skill must exist before AGENTS.md can point to it
- The goal is progressive disclosure: AGENTS.md gives agents enough context to know the ticket system exists and what types/statuses are valid; the skill provides the detailed protocol when they actually need to execute
