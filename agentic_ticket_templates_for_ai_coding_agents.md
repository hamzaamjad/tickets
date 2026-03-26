# Agentic ticket templates for AI coding agents

**The best ticket format for AI coding agents is a YAML-frontmatter markdown file stored in a `.tickets/` directory, combining machine-parseable metadata with human-readable context in a single portable artifact.** This pattern has converged across every major tool ecosystem — Task Master AI (25k+ GitHub stars), GitHub's Spec Kit, Amazon's Strands SOPs, and McKinsey's QuantumBlack framework all independently arrived at the same core structure: structured frontmatter for routing and status, followed by markdown sections that give agents unambiguous instructions. The format works because none of the major AI coding tools (Claude Code, Cursor, Codex) programmatically parse markdown — they inject file contents as natural language context into the LLM, which means any well-structured markdown file is inherently cross-tool portable.

For a power user already running AGENTS.md, SKILL.md, and MCP servers as cross-platform primitives, `.tickets/` becomes the missing execution layer — the bridge between your agent configuration (how agents behave) and your agent work (what agents do).

---

## The schema: one frontmatter block to rule all ticket types

The frontmatter must cover three concerns: **identity** (what is this ticket), **workflow** (where is it in the pipeline), and **relationships** (how does it connect to other work). Every field should be optional except `id`, `title`, `type`, and `status` — this keeps the format lightweight enough for agents to generate tickets without friction while giving humans full expressiveness when needed.

```yaml
---
id: FEAT-003
title: "Add rate limiting to dbt model refresh endpoint"
type: feature          # feature | bug | refactor | chore | custom
status: to-do          # to-do | in-progress | review | done | blocked
priority: high         # critical | high | medium | low
created: 2026-03-26
updated: 2026-03-26
parent: EPIC-001       # parent ticket ID for sub-ticket linking
dependencies:          # tickets that must complete first
  - TASK-001
  - TASK-002
tags: [dbt, api, rate-limiting]
agent_created: false   # true if an AI agent generated this ticket
---
```

**The `type` field drives which markdown sections appear below.** A `feature` ticket needs Context, Requirements, and Acceptance Criteria. A `bug` ticket needs Steps to Reproduce and Expected vs. Actual Behavior. A `refactor` needs Motivation and Scope. But the frontmatter schema stays identical across all types — this is what makes the system parseable. Agents key off `type` to know which template pattern to follow; humans key off `status` and `priority` to know what needs attention.

The `agent_created` boolean is a small but critical field. It signals to human reviewers that the ticket was decomposed by an agent and may need validation before work begins. Task Master AI's complexity scoring (1-10) is worth borrowing as an optional `complexity` field — it helps agents self-assess whether a ticket should be further decomposed.

---

## The four ticket templates adapted for your stack

Each template follows the same structural contract: frontmatter → one-line summary → structured sections → verification commands. The sections vary by type, but the opening and closing patterns stay consistent so agents can reliably navigate any ticket.

### Feature ticket

```markdown
---
id: FEAT-003
title: "Add rate limiting to dbt model refresh endpoint"
type: feature
status: to-do
priority: high
created: 2026-03-26
parent: EPIC-001
dependencies: [TASK-001]
tags: [dbt, api, rate-limiting]
---

# Add rate limiting to dbt model refresh endpoint

## Context
The `/api/refresh` endpoint triggers full dbt model rebuilds against
PostgreSQL. Without rate limiting, concurrent requests can saturate
database connections and cause cascading failures in downstream
dashboards. This supports EPIC-001 (API hardening).

## Requirements
- [ ] Implement token bucket rate limiter as FastAPI middleware
- [ ] Configure per-user limits: 10 requests/minute for refresh endpoints
- [ ] Configure global limits: 50 requests/minute across all users
- [ ] Store rate limit state in Redis (existing instance at `services/redis`)
- [ ] Return 429 with `Retry-After` header when limit exceeded

## File path hints
- `src/api/middleware/rate_limit.py` — new middleware (create)
- `src/api/routes/refresh.py` — apply middleware to refresh routes
- `src/config/limits.py` — rate limit configuration constants
- `tests/api/middleware/test_rate_limit.py` — test suite
- `docker-compose.yml` — verify Redis service is exposed

## Constraints
- Do NOT modify existing dbt model execution logic
- Do NOT add new Python dependencies; use existing `redis` package
- Must maintain backward compatibility with existing API clients
- Rate limiter must not block health check endpoints

## Acceptance criteria
- [ ] 11th request within 60 seconds returns HTTP 429
- [ ] `Retry-After` header value matches actual reset time (±1 second)
- [ ] Rate limit state persists across API server restarts
- [ ] Health check endpoints (`/health`, `/ready`) are exempt
- [ ] All existing tests in `tests/api/` continue to pass

## Verification
```bash
pytest tests/api/middleware/test_rate_limit.py -v
pytest tests/api/ -v --tb=short
ruff check src/api/middleware/rate_limit.py
docker compose exec api python -c "from src.api.middleware.rate_limit import RateLimiter; print('import ok')"
```

## Notes
See `docs/architecture.md` for Redis connection pooling patterns
already established in this project.
```

### Bug ticket

Bug tickets swap Requirements for **reproduction steps** and add the Expected/Actual behavior split — the two things agents need most to localize and fix a defect.

```markdown
---
id: BUG-017
title: "dbt incremental model fails silently on NULL partition key"
type: bug
status: to-do
priority: critical
created: 2026-03-26
tags: [dbt, postgresql, data-integrity]
---

# dbt incremental model fails silently on NULL partition key

## Description
The `orders_incremental` dbt model skips rows where `partition_date`
is NULL instead of raising an error. This causes silent data loss
in downstream reporting tables.

## Steps to reproduce
1. Insert a row into `raw.orders` with `partition_date = NULL`
2. Run `dbt run --select orders_incremental`
3. Query `analytics.orders_incremental` for the inserted row
4. Row is missing — no error in dbt logs

## Expected behavior
dbt run fails with a clear error message identifying the NULL
partition key, or the model handles NULLs explicitly with a
configurable default.

## Actual behavior
Model completes successfully. NULL partition rows are silently
dropped by the `WHERE partition_date > (select max...)` clause.

## File path hints
- `models/staging/orders_incremental.sql` — the incremental model
- `tests/staging/test_orders_incremental.py` — existing tests
- `macros/incremental_helpers.sql` — shared incremental logic

## Root cause hypothesis
The incremental predicate `WHERE partition_date > ...` evaluates
to NULL when `partition_date` is NULL, which SQL treats as FALSE.

## Constraints
- Do NOT change the incremental strategy (keep `merge` strategy)
- Fix must work with PostgreSQL 15+ NULL handling semantics
- Do NOT modify the source table schema

## Acceptance criteria
- [ ] NULL partition keys cause dbt to raise an explicit error
- [ ] Error message includes the model name and column name
- [ ] Existing non-NULL rows are unaffected
- [ ] dbt test suite passes: `dbt test --select orders_incremental`

## Verification
```bash
dbt run --select orders_incremental 2>&1 | grep -i "null\|error"
dbt test --select orders_incremental
pytest tests/staging/test_orders_incremental.py -v
```
```

### Refactor ticket

Refactor tickets emphasize **zero behavior change** and define scope through explicit file listings rather than abstract descriptions.

```markdown
---
id: REFAC-008
title: "Extract SQL query builder into shared module"
type: refactor
status: to-do
priority: medium
created: 2026-03-26
tags: [tech-debt, sql, dry]
---

# Extract SQL query builder into shared module

## Motivation
Raw SQL query construction is duplicated across 5 API route files,
each building WHERE clauses and pagination independently. This
creates inconsistency in SQL injection protection and makes
pagination bugs harder to fix.

## Scope
Files with duplicated query building logic:
- `src/api/routes/orders.py` (lines 45-89)
- `src/api/routes/customers.py` (lines 32-71)
- `src/api/routes/products.py` (lines 28-65)
- `src/api/routes/invoices.py` (lines 41-80)
- `src/api/routes/shipments.py` (lines 35-72)

## Target structure
- Create `src/api/utils/query_builder.py` — shared query builder
- Create `tests/api/utils/test_query_builder.py` — unit tests

## Acceptance criteria
- [ ] All 5 route files use shared `QueryBuilder` class
- [ ] Zero behavior change — all existing API tests pass unchanged
- [ ] Shared module has >95% test coverage
- [ ] `ruff check` and `mypy` pass with no new warnings
- [ ] No new SQL injection vectors (parameterized queries only)

## Constraints
- Pure refactor — NO feature changes, NO API contract changes
- Do NOT rename any existing public functions or endpoints
- Do NOT change database query results or response shapes

## Verification
```bash
pytest tests/ -v --tb=short
ruff check src/api/utils/query_builder.py
mypy src/api/utils/query_builder.py --strict
pytest tests/api/ -v  # full API test suite unchanged
```
```

---

## Directory structure and the file organization protocol

The `.tickets/` directory lives at the repository root alongside your existing `AGENTS.md` and `CLAUDE.md`. The organizational principle is **flat with semantic naming** — not nested status directories. Status lives in frontmatter, not in the filesystem, because moving files between directories breaks git blame and makes agent-generated cross-references fragile.

```
your-project/
├── AGENTS.md                          # cross-tool agent instructions
├── CLAUDE.md                          # Claude Code-specific (or symlink)
├── .cursor/rules/                     # Cursor-specific scoped rules
├── .claude/skills/                    # reusable agent skills
├── .tickets/
│   ├── _templates/                    # ticket templates by type
│   │   ├── feature.md
│   │   ├── bug.md
│   │   ├── refactor.md
│   │   └── chore.md
│   ├── EPIC-001_api-hardening.md      # epic/parent tickets
│   ├── FEAT-003_rate-limiting.md      # feature tickets
│   ├── BUG-017_null-partition.md      # bug tickets
│   ├── REFAC-008_query-builder.md     # refactor tickets
│   ├── TASK-001_redis-config.md       # sub-tasks (agent-generated)
│   └── TASK-002_middleware-base.md    # sub-tasks (agent-generated)
├── src/
├── models/                            # dbt models
├── tests/
└── docker-compose.yml
```

**The naming convention is `TYPE-NNN_kebab-slug.md`** where TYPE is a 2-6 character prefix (`FEAT`, `BUG`, `REFAC`, `TASK`, `EPIC`, `CHORE`), NNN is a zero-padded sequential number, and the slug is a human-scannable description. This convention comes from Roo Commander's MDTM pattern and has proven the most reliable across tools — the TYPE prefix lets agents filter by work category using simple glob patterns (`ls .tickets/BUG-*.md`), while the slug gives humans instant context when browsing the directory.

**IDs must be globally unique within the `.tickets/` directory.** When agents create sub-tickets, they use `TASK-NNN` with the next available number and set `parent: FEAT-003` in frontmatter to establish the relationship. This is intentionally simpler than Task Master AI's dotted notation (`1.1.1`) because dotted IDs embed hierarchy in the identifier itself, which breaks when tickets get reparented or reorganized.

---

## How agents decompose work into sub-tickets

Bidirectional ticket creation is the system's most powerful capability. Your `AGENTS.md` (or a SKILL.md file) should include explicit instructions for how agents create and consume tickets. Here is the decomposition protocol that works across Claude Code, Cursor, and Codex:

```markdown
## Ticket System (.tickets/)

When assigned a ticket from `.tickets/`, follow this protocol:

1. **Read the ticket** completely before writing any code
2. **Check dependencies** — if any dependency has status != "done", STOP
   and report the blocker
3. **Assess complexity** — if the ticket requires changes to 4+ files or
   has 5+ acceptance criteria, decompose into sub-tickets before coding
4. **Decompose** by creating new TASK-NNN files in `.tickets/` with:
   - `parent:` pointing to the original ticket ID
   - `dependencies:` reflecting execution order
   - `agent_created: true`
   - Each sub-ticket targeting 1-2 files and ≤3 acceptance criteria
5. **Update the parent ticket** status to "in-progress" and add a
   `## Sub-tickets` section listing the generated TASK IDs
6. **Execute sub-tickets** in dependency order, updating each to "done"
   when its verification commands pass
7. **Mark parent done** only when ALL sub-tickets are done and parent
   acceptance criteria pass
```

**The key insight from QuantumBlack's research is that agents should not self-orchestrate the decomposition sequence.** Their 2026 study found that agents "routinely skipped steps, created circular dependencies, or got stuck in analysis loops" when given full orchestration autonomy. The protocol above is deliberately deterministic — read, check, assess, decompose, execute, verify. The agent makes creative decisions within each step but follows the step order rigidly.

**Agent-generated sub-tickets should be smaller than human-written tickets.** The sweet spot from Task Master AI's community data is **1-2 files changed per sub-ticket with ≤3 acceptance criteria**. This keeps each unit of work within a single agent context window without compaction, and makes verification fast. A feature ticket that touches 8 files should decompose into 4-5 sub-tickets, not 8 — group by logical concern, not by file.

When an agent creates sub-tickets, the parent ticket gets a tracking section appended:

```markdown
## Sub-tickets
| ID | Title | Status |
|---|---|---|
| TASK-001 | Configure Redis connection pool | done |
| TASK-002 | Implement base middleware class | in-progress |
| TASK-003 | Add rate limit logic and tests | to-do |
| TASK-004 | Wire middleware to refresh routes | to-do |
```

This table is the one place where duplication between frontmatter and body is acceptable — it gives humans a dashboard view of decomposed work without requiring them to open every sub-ticket file.

---

## Cross-tool portability: why this format works everywhere

The three major AI coding tools consume workspace files through fundamentally similar mechanisms, which is why the `.tickets/` format works without modification across all of them.

**Claude Code** discovers `CLAUDE.md` files hierarchically (upward from CWD at startup, lazy-loading child directories on demand) and injects their contents as system prompt context. It also reads `AGENTS.md` as a fallback. Crucially, Claude Code does not programmatically parse YAML frontmatter — the entire file contents are fed to the LLM as natural language. This means your ticket files work perfectly when referenced via `@.tickets/FEAT-003_rate-limiting.md` in a prompt, or when an agent reads them using file tools. Keep individual tickets under **200 lines** — Anthropic's data shows >92% instruction adherence under 200 lines versus just 71% beyond 400 lines.

**Cursor** uses `.cursor/rules/*.mdc` files with YAML frontmatter (the only tool that programmatically parses frontmatter for its `description`, `globs`, and `alwaysApply` fields). But Cursor also reads `AGENTS.md` and `CLAUDE.md` at the project root. For ticket consumption, agents in Cursor's agent mode read `.tickets/` files directly through file operations — the tickets don't need `.mdc` format because they're not rules, they're task inputs. You can create a single Cursor rule that teaches the agent about the ticket system:

```yaml
---
description: "When the user asks to work on a ticket, read the .tickets/ directory"
alwaysApply: false
---
Read tickets from `.tickets/` directory. Follow the decomposition protocol
in AGENTS.md. Update ticket status in frontmatter as you work.
```

**Codex** builds an instruction chain by walking from git root to CWD, loading `AGENTS.md` (or `AGENTS.override.md`) at each level. Like Claude Code, it injects file contents as user-role messages — no programmatic parsing. Codex's 32KB default size limit (`project_doc_max_bytes`) applies to instruction files, not to files the agent reads during execution, so ticket size is not constrained by this limit.

**The portability principle: YAML frontmatter is parsed by the LLM, not by the tool runtime.** Since all three tools ultimately feed file contents to a language model, and language models reliably parse YAML frontmatter as structured data, the format is inherently portable. You don't need tool-specific syntax — you need clear, consistent structure that any frontier model can interpret.

---

## Seven anti-patterns that make tickets unactionable

Research across GitHub's analysis of 2,500+ agent files, Addy Osmani's spec-writing framework, and QuantumBlack's production findings reveals consistent failure modes.

**The vague spec** is the most common killer. "Improve the API" gives agents nothing to anchor on. Every ticket needs a concrete noun (what changes), a concrete verb (create, modify, delete, move), and a concrete target (specific files, endpoints, or behaviors). For your dbt/PostgreSQL stack, "Fix the data pipeline" is vague; "Add NOT NULL constraint validation to the `orders_incremental` model's `partition_date` column in `models/staging/orders_incremental.sql`" is actionable.

**Context overload** ranks second. Anthropic's research confirms a "curse of instructions" — model adherence drops significantly as directive count increases. The fix is decomposition: one ticket per concern, with **≤5 acceptance criteria per ticket**. If you're writing more than 5, the ticket needs splitting.

**Missing constraints cause scope creep** that's expensive to unwind. Agents will "try to complete a task at any cost," including modifying CI configuration, adding dependencies, or refactoring adjacent code. Every ticket needs an explicit Constraints section. For your stack, common constraints include: "Do NOT modify `docker-compose.yml`", "Do NOT add new PyPI dependencies", "Do NOT change dbt model materializations."

**Stale file path hints poison agent context.** When `src/auth/handlers.py` gets renamed but the ticket still references it, the agent wastes tokens searching for a nonexistent file, then either hallucinates a fix or asks for clarification. Reference capabilities ("the authentication middleware") rather than exact paths when the codebase is volatile, or commit to keeping path hints current.

**No verification commands** means no feedback loop. The most effective tickets include exact `pytest`, `ruff`, `dbt test`, or `docker compose exec` commands that the agent can run to self-check. Without these, agents have no way to confirm their work meets criteria — they're flying blind.

**Contradictory instructions across files** cause unpredictable behavior. If your `AGENTS.md` says "always use `pathlib`" but a ticket's Notes section shows an `os.path` example, the agent may follow either one. Treat your instruction surface (`AGENTS.md` + `CLAUDE.md` + `.cursor/rules/` + ticket files) as a single system and audit for contradictions.

**Correcting mid-session instead of restarting** is a workflow anti-pattern rather than a template one, but it matters. When an agent goes down the wrong path on a ticket, the incorrect reasoning stays in context and actively degrades subsequent attempts. Revert changes, clarify the ticket, and start a fresh session — the ticket file becomes your reset point.

---

## Integrating tickets with your existing architecture

Your cross-tool stack (AGENTS.md + SKILL.md + MCP servers) provides three natural integration points for the ticket system.

**AGENTS.md** is where the ticket protocol lives. Add a section describing the `.tickets/` directory, the naming convention, the decomposition rules, and the status lifecycle. This is the single instruction that all three tools (Claude Code, Cursor, Codex) will read before touching any ticket. Keep this section under 30 lines — link to a `SKILL.md` for the full protocol details.

**A `ticket-workflow` SKILL.md** handles the detailed decomposition and execution protocol. Store it at `.claude/skills/ticket-workflow/SKILL.md` (Claude Code) and symlink or copy to `.agents/skills/ticket-workflow/SKILL.md` (Codex). The skill's `description` field should read: "Use when working on structured tickets from the .tickets/ directory. Handles ticket reading, decomposition into sub-tickets, status updates, and verification." This progressive disclosure pattern keeps AGENTS.md lean while giving agents full protocol access when they need it.

**An MCP server for ticket management** is the highest-leverage integration. Build a lightweight MCP server (Python, since it's your stack) that exposes tools like `list_tickets(status?, type?)`, `read_ticket(id)`, `update_status(id, status)`, `create_ticket(type, title, parent?)`, and `next_ticket()`. This gives agents structured access to the ticket system without relying on file parsing, works identically across Claude Code, Cursor, and Codex (all support MCP), and lets you add validation logic (circular dependency detection, ID uniqueness, status transition rules) that pure file-based systems can't enforce. Task Master AI's MCP server with **36+ tools** and the Shrimp Task Manager MCP server both prove this pattern scales.

The MCP approach also solves the **ID generation problem** cleanly. Instead of agents guessing the next available ID by scanning filenames, the `create_ticket` tool returns a guaranteed-unique ID. This eliminates race conditions when multiple agents decompose tickets in parallel.

---

## Conclusion

The `.tickets/` directory pattern fills a specific gap in the agentic coding stack: **persistent, structured, version-controlled units of work** that both humans and agents can create, read, and execute. The template design converges on YAML frontmatter (4 required fields: `id`, `title`, `type`, `status`) plus typed markdown sections that vary by ticket type but follow a consistent contract (Context → Requirements → Constraints → Acceptance Criteria → Verification).

Three design decisions matter most. First, **status lives in frontmatter, not in directory structure** — this preserves git history and makes cross-references stable. Second, **sub-tickets reference parents via ID, not via filesystem nesting** — this keeps the directory flat and scannable while supporting arbitrary decomposition depth. Third, **verification commands are non-negotiable** — they're what close the loop between "agent thinks it's done" and "work is actually done."

The format's portability comes not from any tool-specific feature but from a structural reality: every major AI coding tool feeds file contents to a language model as natural language context. Well-structured markdown with YAML frontmatter is the universal interchange format for this architecture. Pair it with an MCP server for validation and ID management, and you have a ticket system that's as rigorous as Jira but lives where agents actually work — in the codebase.