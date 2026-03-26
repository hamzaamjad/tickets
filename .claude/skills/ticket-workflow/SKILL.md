---
name: ticket-workflow
description: "Manage structured tickets in the .tickets/ directory. Use when working on a ticket, creating a ticket, decomposing tickets into sub-tickets, updating ticket status, or verifying ticket completion. Handles the full ticket lifecycle: reading, dependency checking, complexity assessment, decomposition, execution, and verification."
---

# Ticket Workflow Protocol

Complete procedural guide for executing structured tickets from `.tickets/`.

## Naming Convention

Files: `TYPE-NNN_kebab-slug.md` (e.g., `FEAT-001_add-login.md`)

| Prefix | Type     | Use case                                    |
|--------|----------|---------------------------------------------|
| `FEAT` | feature  | New functionality                           |
| `BUG`  | bug      | Defect fix                                  |
| `REFAC`| refactor | Structural improvement, zero behavior change|
| `CHORE`| chore    | Maintenance, CI, docs, tooling              |
| `TASK` | task     | Agent-generated sub-ticket                  |
| `EPIC` | epic     | Parent ticket grouping related work         |

## Status Lifecycle

`to-do` → `in-progress` → `review` → `done`

A ticket may also be `blocked` at any point.

## Execution Protocol

Follow these steps in exact order when assigned a ticket:

### Step 1: Read the ticket

Read the full ticket file before writing any code. Understand requirements, constraints, acceptance criteria, and verification commands.

### Step 2: Check dependencies

Inspect the `dependencies` array in frontmatter. If any dependency has status != `done`, STOP and report the blocker to the user.

### Step 3: Assess complexity

If the ticket requires changes to 4+ files or has 5+ acceptance criteria, decompose into sub-tickets before coding. Proceed to Step 4.

If the ticket is small enough, skip to Step 5.

### Step 4: Decompose into sub-tickets

Create new `TASK-NNN` files in `.tickets/` with:
- `parent:` pointing to the original ticket ID
- `dependencies:` reflecting execution order between sub-tickets
- `agent_created: true`
- Each sub-ticket targeting 1-2 files and ≤3 acceptance criteria

Read the appropriate template from `.tickets/_templates/` before creating each sub-ticket.

Update the parent ticket: set status to `in-progress` and append a tracking table:

```markdown
## Sub-tickets
| ID       | Title         | Status |
|----------|---------------|--------|
| TASK-001 | [description] | to-do  |
| TASK-002 | [description] | to-do  |
```

Execute sub-tickets in dependency order. Update each to `done` when its verification commands pass.

### Step 5: Execute

- Set ticket status to `in-progress` and update the `updated` date in frontmatter
- Read all files referenced in the ticket's file path hints before making changes
- Implement the requirements
- Stay within the ticket's constraints — do not add features or improvements beyond scope

### Step 6: Verify

Run every command in the ticket's `## Verification` section. All must pass. If any fail, fix the issue and re-run.

### Step 7: Mark done

- Set ticket status to `done` and update the `updated` date
- If this ticket has a parent, check whether ALL sibling sub-tickets are also `done`
- Mark the parent `done` only when ALL sub-tickets are `done` and the parent's acceptance criteria pass

## Creating New Tickets

Use the naming convention `TYPE-NNN_kebab-slug.md`. Choose the next available number for the type prefix.

Read the matching template from `.tickets/_templates/` for the ticket type before creating the file. Available templates:
- `.tickets/_templates/feature.md`
- `.tickets/_templates/bug.md`
- `.tickets/_templates/refactor.md`
- `.tickets/_templates/chore.md`

Required frontmatter fields: `id`, `title`, `type`, `status`.

## Quality Rules

- Keep tickets under 200 lines
- Maximum 5 acceptance criteria per ticket — decompose if more are needed
- Every ticket must include a `## Verification` section with runnable commands
- Include a `## Constraints` section to prevent scope creep
- Use concrete nouns, verbs, and file paths — never vague instructions
- File path hints should reference capabilities ("the auth middleware") when paths are volatile
- Do NOT self-orchestrate decomposition sequences — follow the step order above rigidly
