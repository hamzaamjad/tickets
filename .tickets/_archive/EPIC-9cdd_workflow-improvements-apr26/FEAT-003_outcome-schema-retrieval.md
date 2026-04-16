---
id: FEAT-003
title: "## Outcome section + Lane A archive retrieval"
type: feature
status: done
priority: medium
created: 2026-04-16
updated: 2026-04-16
parent: EPIC-9cdd
dependencies: [REFAC-001]
tags: [ticket-workflow, archive, retrieval, outcome]
agent_created: false
complexity: 6
---

# `## Outcome` section + Lane A archive retrieval (D1)

## Context

Closed tickets land in `.tickets/_archive/` and are immediately forgotten.
Nothing tells a future agent how to mine them for the invariants, gotchas,
or tested patterns they contain. R4 recommends a two-part fix: standardize
a dense `## Outcome` block at closure time (so each archived ticket is
self-summarizing), and ship a thin retrieval script that surfaces matching
Outcome blocks at epic-creation time.

This ticket ships **Lane A only** — the schema reference, the SKILL.md
wiring (Step 7 addendum + `## Querying past work` section), and a
Lane A-only shell script. The semantic Lane B is a separate deferred
ticket, gated by the ship trigger documented in `references/outcome-schema.md`.

The plan splits D into two sub-tickets so this one stays bounded; keeping
both lanes in a single ticket would push it past the complexity tier
threshold landed in REFAC-001, and Lane B's precondition (archive size ≥
~2,000 chunks, or two logged Lane A runs that return ≥ 5 irrelevant
matches) is not yet met.

Source: `docs/implementation_plan.md` §2 "[FEAT] `## Outcome` section +
Lane A archive retrieval (D1)". Primary source: `docs/research/r4-archive-knowledge-retrieval.md`
§"Structured outcome section proposal" and §"Upgrade path as the archive
grows". Resolves adversarial review §3 "Step 7 sub-step ambiguity", §3
"scale horizon not named", §4 "`--help` passes with a stub".

## Requirements

- [ ] Create `.claude/skills/ticket-workflow/references/outcome-schema.md`
      with:
      - The `## Outcome` schema reproduced **verbatim** from R4
        §"Structured outcome section proposal" — the seven bolded
        subsections in this order: `**Summary:**`, `**Key decisions:**`,
        `**Constraints & invariants discovered (keep):**`,
        `**Implementation notes (high signal only):**`, `**Verification:**`,
        `**Risk / regression surface:**`, `**Retrieval tags:**`. Target
        length: **120–200 words / ≤ 8 short bullets**.
      - The filled "request-scoped cache" example from R4
        §"Filled example" (lines 210–240).
      - One closing paragraph naming the scale horizon for Lane B:
        *"When `.tickets/_archive/` exceeds ~2,000 `## Outcome` chunks
        (≈ 200 tickets at ~10 bullets each), Lane A precision will
        degrade; at that point ship Lane B per `docs/implementation_plan.md`
        §4 trigger. Until then, Lane A alone is sufficient. See
        `docs/research/r4-archive-knowledge-retrieval.md` §'Upgrade path
        as the archive grows'."*
- [ ] Modify `.claude/skills/ticket-workflow/SKILL.md` Step 7:
      - Insert a new **first sub-step** before "Set `status: done`":
        *"Append an `## Outcome` section to the ticket using the schema in
        `references/outcome-schema.md`."*
      - Immediately after that sub-step, add the commit-policy
        clarification: *"The Outcome append is staged in the same commit
        as `status: done` (the mark-done commit). The 'separate from
        implementation work' rule refers to **production code changes**,
        not ticket metadata written at closure."*
- [ ] Append a new top-level section `## Querying past work` to
      `.claude/skills/ticket-workflow/SKILL.md` **below** the existing
      `## Epic Closure Ticket` section. Content:
      - One paragraph directing agents to run
        `bash .claude/skills/ticket-workflow/scripts/archive-search.sh '<short epic pitch>'`
        before creating a new epic that touches unfamiliar subsystems, and
        to inject top matches' `## Outcome` blocks into the new epic's
        Context section.
- [ ] Create `.claude/skills/ticket-workflow/scripts/archive-search.sh`
      implementing **Lane A only**, with:
      - Shebang `#!/usr/bin/env bash` and executable bit set.
      - `set -euo pipefail`.
      - `--help` prints usage: query syntax, `--type`, `--complexity`,
        `--tags` flag descriptions, **and** one line explicitly stating
        *"Lane B (`--semantic`) is not yet available; see
        `.claude/skills/ticket-workflow/references/outcome-schema.md` for
        the ship trigger."*
      - Default behavior: `rg -l --type md` over `.tickets/_archive/` to
        locate files whose YAML frontmatter or `## Outcome` block matches
        the query. Flag filters apply as frontmatter post-filters
        (`--type` matches the `type:` line; `--complexity` matches an
        integer in the `complexity:` line; `--tags` matches any token in
        the `tags:` list).
      - Output: matching paths **+ their `## Outcome` section snippets**
        only — never the full ticket body.
      - **Empty archive handling:** if `.tickets/_archive/` does not exist
        or contains zero matches, print `no matches` to stdout and exit 0.
        Smoke tests must not false-fail on fresh workspaces.

## File path hints

- `.claude/skills/ticket-workflow/references/outcome-schema.md` — create.
- `.claude/skills/ticket-workflow/scripts/archive-search.sh` — create, set
  executable bit.
- `.claude/skills/ticket-workflow/SKILL.md` — modify (Step 7 first
  sub-step + commit-policy note; new `## Querying past work` section).
- `docs/research/r4-archive-knowledge-retrieval.md` — **read-only source**,
  do not edit.

## Constraints

- Lane A only. Do NOT add a `--semantic` code path in this ticket; ship
  only the `--help` mention of Lane B as a future option.
- Do NOT write any persistent index to disk (R4 forbids this binding
  constraint; Lane B's eventual implementation honors it by building
  in-session only).
- Do NOT include the full ticket body in search output — only the file
  path and the `## Outcome` snippet. Privacy / signal-to-noise discipline.
- The Step 7 commit-policy clarification must use the phrase
  `production code changes` verbatim — AC-5 asserts on it.
- Do NOT fail on an empty archive; `no matches` + exit 0 is required so
  fresh workspaces bootstrap without a stub archive.
- Do NOT add the script to any CI — it is an opt-in agent tool, not a
  test.

## Acceptance criteria

- [ ] Schema shape: seven bolded subsections in the correct order.
      `[ -f .claude/skills/ticket-workflow/references/outcome-schema.md ]`
      **and**
      `grep -cE '^\*\*(Summary|Key decisions|Constraints & invariants discovered \(keep\)|Implementation notes \(high signal only\)|Verification|Risk / regression surface|Retrieval tags):\*\*'
      .claude/skills/ticket-workflow/references/outcome-schema.md`
      returns exactly `7`.
- [ ] Scale-horizon sentence present:
      `grep -cE '2,?000 \`?## Outcome\`? chunks'
      .claude/skills/ticket-workflow/references/outcome-schema.md` returns ≥ 1.
- [ ] `--help` works and mentions Lane B's unavailability:
      `bash .claude/skills/ticket-workflow/scripts/archive-search.sh --help`
      exits 0 **and** its output contains both `--semantic` and the
      `not yet available` message.
- [ ] End-to-end fixture test passes against a disposable archive:
      ```bash
      TMP=$(mktemp -d); mkdir -p "$TMP/.tickets/_archive/EPIC-test_x"; \
        printf -- '---\ntype: feature\ntags: [cache]\n---\n## Outcome\n**Summary:** fixture.\n**Retrieval tags:** cache\n' \
          > "$TMP/.tickets/_archive/EPIC-test_x/FEAT-001_y.md"; \
        (cd "$TMP" && bash "$OLDPWD/.claude/skills/ticket-workflow/scripts/archive-search.sh" cache) \
          | grep -q 'FEAT-001_y.md'
      ```
- [ ] SKILL.md integration:
      `grep -cE '^## Querying past work$' .claude/skills/ticket-workflow/SKILL.md`
      returns `1`, **and**
      `grep -c 'references/outcome-schema.md' .claude/skills/ticket-workflow/SKILL.md`
      returns ≥ 1, **and**
      `grep -c 'production code changes' .claude/skills/ticket-workflow/SKILL.md`
      returns ≥ 1.

## Verification

```bash
SCHEMA=.claude/skills/ticket-workflow/references/outcome-schema.md
SCRIPT=.claude/skills/ticket-workflow/scripts/archive-search.sh
SKILL=.claude/skills/ticket-workflow/SKILL.md

# AC-1: schema shape
[ -f "$SCHEMA" ]
[ "$(grep -cE '^\*\*(Summary|Key decisions|Constraints & invariants discovered \(keep\)|Implementation notes \(high signal only\)|Verification|Risk / regression surface|Retrieval tags):\*\*' "$SCHEMA")" = "7" ]

# AC-2: scale-horizon sentence
grep -qE '2,?000 `?## Outcome`? chunks' "$SCHEMA"

# AC-3: --help is correct
[ -x "$SCRIPT" ] || chmod +x "$SCRIPT"
OUT=$(bash "$SCRIPT" --help)
echo "$OUT" | grep -q '\-\-semantic'
echo "$OUT" | grep -q 'not yet available'

# AC-4: end-to-end fixture
OLDPWD_ABS=$(pwd)
TMP=$(mktemp -d)
mkdir -p "$TMP/.tickets/_archive/EPIC-test_x"
printf -- '---\ntype: feature\ntags: [cache]\n---\n## Outcome\n**Summary:** fixture.\n**Retrieval tags:** cache\n' \
  > "$TMP/.tickets/_archive/EPIC-test_x/FEAT-001_y.md"
(cd "$TMP" && bash "$OLDPWD_ABS/$SCRIPT" cache) | grep -q 'FEAT-001_y.md'
rm -rf "$TMP"

# AC-5: SKILL.md integration
[ "$(grep -cE '^## Querying past work$' "$SKILL")" = "1" ]
[ "$(grep -c 'references/outcome-schema.md' "$SKILL")" -ge 1 ]
[ "$(grep -c 'production code changes' "$SKILL")" -ge 1 ]

# Sanity: Step 7 still leads with the Outcome append
awk '/^### Step 7: Mark done$/,/^## /' "$SKILL" | head -20 | grep -q '## Outcome'
```

## Notes

- The fixture test deliberately builds its archive on tmpfs so it does not
  pollute the real `.tickets/_archive/` or require a pre-seeded corpus.
  Empty-archive handling (`no matches` + exit 0) is what prevents this
  fixture pattern from false-failing on a cold workspace.
- The script's query semantics are deliberately minimal: plain `rg`
  substring match across frontmatter and `## Outcome` block. This is
  R4's "lexical search is sufficient below ~2,000 chunks" recommendation.
- `scripts/` directory for the skill is new — create it with
  `mkdir -p` inside the script's implementation step. If another skill
  ever shares the directory, keep `archive-search.sh` self-contained
  (no shared helpers).
- The commit-policy clarification is intentionally narrow: "production
  code changes" is the exact phrase a future agent will grep for. The
  Outcome append writes ticket metadata, not code.
