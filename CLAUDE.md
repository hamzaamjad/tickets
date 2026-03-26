# CLAUDE.md

Claude Code-specific configuration for this workspace.

## Instructions

All cross-tool agent instructions (ticket system, coding standards, decomposition protocol) live in [AGENTS.md](./AGENTS.md). Read and follow AGENTS.md for all task execution.

This file contains only Claude Code-specific overrides or additions below.

## Claude Code Specifics

- When working on a ticket, read the full ticket file before writing any code
- Use the `.tickets/_templates/` directory to create new tickets matching the correct type
- Update ticket frontmatter `status` and `updated` fields as work progresses
- After completing a ticket, run all verification commands from the ticket's `## Verification` section
