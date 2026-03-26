---
id: FEAT-001
title: "Create ticket-workflow SKILL.md for the .tickets/ system"
type: feature
status: done
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
# File exists and has frontmatter
head -5 .claude/skills/ticket-workflow/SKILL.md | grep -q "^---"
# File is under 200 lines
test $(wc -l < .claude/skills/ticket-workflow/SKILL.md) -le 200
# File contains key protocol elements
grep -q "dependencies" .claude/skills/ticket-workflow/SKILL.md
grep -q "decompose" .claude/skills/ticket-workflow/SKILL.md
grep -q "Verification" .claude/skills/ticket-workflow/SKILL.md
grep -q "_templates" .claude/skills/ticket-workflow/SKILL.md
echo "All checks passed"
```

## Notes
- Reference document: `agentic_ticket_templates_for_ai_coding_agents.md` (section: "Integrating tickets with your existing architecture") describes the SKILL.md pattern
- The skill description field should read something like: "Use when working on structured tickets from the .tickets/ directory. Handles ticket reading, decomposition into sub-tickets, status updates, and verification."
- Keep the skill focused on *procedure* (how to work on tickets), not *policy* (coding standards) — AGENTS.md retains ownership of general coding standards
