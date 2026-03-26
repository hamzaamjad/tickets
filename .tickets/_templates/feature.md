---
id: FEAT-XXX
title: ""
type: feature
status: to-do
priority: medium         # critical | high | medium | low
created: YYYY-MM-DD
updated: YYYY-MM-DD
parent:                  # parent ticket ID (e.g., EPIC-001)
dependencies: []         # ticket IDs that must complete first
tags: []
agent_created: false
complexity:              # 1-10, optional — helps agents decide whether to decompose
---

# [Title]

## Context
[Why does this feature exist? What problem does it solve? Link to parent epic
or initiative if applicable.]

## Requirements
- [ ] [Concrete, verifiable requirement]
- [ ] [One requirement per checkbox]
- [ ] [Each should map to a testable outcome]

## File path hints
- `path/to/file.py` — [what to do: create | modify | delete]

## Constraints
- Do NOT [specific thing to avoid]
- Must [specific invariant to maintain]

## Acceptance criteria
- [ ] [Observable outcome with concrete values]
- [ ] [Keep to 5 or fewer — decompose if more are needed]

## Verification
```bash
# Commands the agent should run to self-check
pytest tests/path/to/test_file.py -v
```

## Notes
[Optional: links to docs, architecture decisions, related discussions.]
