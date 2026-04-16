# AGENTS.md

Cross-tool agent instructions. Read by Claude Code, Cursor, and Codex at session start.

## Ticket System (.tickets/)

This workspace uses a structured ticket system. All work — features, bugs, refactors, chores — is tracked as YAML-frontmatter markdown files in `.tickets/`.

| Prefix  | Type     | Use case                                     |
|---------|----------|----------------------------------------------|
| `FEAT`  | feature  | New functionality                            |
| `BUG`   | bug      | Defect fix                                   |
| `REFAC` | refactor | Structural improvement, zero behavior change |
| `CHORE` | chore    | Maintenance, CI, docs, tooling               |
| `TASK`  | task     | Agent-generated sub-ticket                   |
| `EPIC`  | epic     | Parent grouping related work                 |

Status lifecycle: `to-do` → `in-progress` → `done`. Tickets may also be `blocked`.

For the full protocol — ticket creation, ID assignment, epic branching, worktree setup, execution, decomposition, verification, and archival — read the `ticket-workflow` skill before beginning any ticket work.

## Coding Standards

- Do not add features, refactoring, or improvements beyond ticket scope.
- Do not add error handling for scenarios that cannot happen.
- Do not create abstractions for one-time operations.
- Run verification commands before marking a ticket done.
