---
id: BUG-XXX
title: ""
type: bug
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

## Description
[What is broken? What is the user-visible impact?]

## Steps to reproduce
1. [Exact step]
2. [Exact step]
3. [Observe the defect]

## Expected behavior
[What should happen.]

## Actual behavior
[What happens instead.]

## File path hints
- `path/to/file.py` — [suspected location of the defect]

## Root cause hypothesis
[Optional but valuable: your best guess at why this happens.
Agents use this to narrow their search.]

## Constraints
- Do NOT [specific thing to avoid]
- Fix must [specific invariant to maintain]

## Acceptance criteria
- [ ] [The defect no longer occurs under the reproduction steps]
- [ ] [Regression test exists]
- [ ] [Existing tests continue to pass]

## Verification
```bash
# Commands to confirm the fix
pytest tests/path/to/test_file.py -v
```

## Notes
[Optional: logs, screenshots, related issues.]
