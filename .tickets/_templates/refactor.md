---
id: REFAC-XXX
title: ""
type: refactor
status: to-do
priority: medium         # critical | high | medium | low
created: YYYY-MM-DD
updated: YYYY-MM-DD
parent:                  # parent ticket ID if applicable
dependencies: []
tags: []
agent_created: false
complexity:              # 1-10, optional
---

# [Title]

## Motivation
[Why is this refactor needed? What pain does the current code cause?
Be specific — "tech debt" alone is not sufficient.]

## Scope
Files with the code to refactor:
- `path/to/file.py` (lines NN-MM) — [what's wrong]

## Target structure
- Create `path/to/new_module.py` — [purpose]
- Modify `path/to/existing.py` — [what changes]

## Acceptance criteria
- [ ] Zero behavior change — all existing tests pass unchanged
- [ ] [Structural goal achieved]
- [ ] Linter and type checker pass with no new warnings

## Constraints
- Pure refactor — NO feature changes, NO API contract changes
- Do NOT rename any existing public functions or endpoints
- Do NOT change return types or response shapes

## Verification
```bash
# Full test suite must pass unchanged
pytest tests/ -v --tb=short
ruff check path/to/new_module.py
```

## Notes
[Optional: links to design docs, prior discussions.]
