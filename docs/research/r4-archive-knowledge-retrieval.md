# Archive as Active Knowledge Base for a Git-Native Ticket Archive

## Retrieval mechanism comparison

A file-based archive in a git repo is an unusually friendly retrieval target: it’s already text, already structured, and already organized by epic. That means you can get surprisingly far with “boring” lexical tools, and you only need embeddings when you want *semantic* similarity (e.g., “this smells like that auth edge-case we hit last quarter”) rather than exact-string matches.

### Practical comparison for 50–200 epics

| Mechanism | V1 effort (hrs) | Query latency (50–200 epics) | Retrieval quality for your two use cases | Notes on fit in a git-only world |
|---|---:|---:|---|---|
| ripgrep/grep over frontmatter + body | 0.5–2 | ~0.05–0.5s | **High** for regressions when you know keywords / identifiers; **Medium** for “similar but different” planning | Best baseline. ripgrep is optimized for scanning repos quickly; practical even on large trees. citeturn11view0 |
| `git grep` on tracked files | 0.25–1 | ~0.05–0.5s | Similar to ripgrep; slightly better when you want “tracked-only” and consistent behavior | Git’s native file search over tracked content; pairs cleanly with other git workflows. citeturn4search32 |
| `git log --grep` (commit messages) + `git log -S` (“pickaxe”) | 1–3 | ~0.2–2s (depends on repo size) | **Medium** for regressions (“when did we last touch X?”); **Low–Medium** for planning (commit messages rarely capture deep constraints) | `--grep` searches commit messages; `-S` finds commits that added/removed a string in diffs—useful for tracking when patterns changed. citeturn20view0 |
| On-demand embeddings (local sentence-transformers) + in-memory brute-force similarity | 2–4 | Build: ~2–15s; Query: ~10–200ms | **High** for planning (“similar epics/tickets”); **Medium** for regressions (semantic match can miss exact identifiers) | Local models can be fast on CPU; many semantic-search SBERT models are documented around hundreds of encodes/sec on CPU, which makes indexing a few thousand chunks feasible in-session. citeturn16view0 |
| Embeddings via API calls (OpenAI/Voyage/Cohere) + in-memory similarity | 2–4 | Build: seconds–minutes (network + rate limits); Query: 100ms–1s | **High** semantic quality potential; best when you don’t want local model dependencies | Quality can be strong (e.g., OpenAI’s embedding models report benchmark gains on MIRACL/MTEB). But network + cost + key management adds friction. citeturn19view0turn5search2turn4search3 |
| Ephemeral vector index rebuilt on demand (FAISS) | 3–6 | Build: ~1–10s; Query: ~1–20ms | **High** for planning at scale; **Medium** for regressions | FAISS is built for efficient similarity search and can scale beyond RAM if needed, but your near-term corpus is small enough that simple indexes work. citeturn21view0 |
| Ephemeral vector index rebuilt on demand (ChromaDB in-memory) | 3–6 | Build: ~2–15s; Query: ~5–50ms | Similar to FAISS; adds metadata filtering ergonomics | Chroma’s `EphemeralClient` explicitly stores data in memory and does not persist to disk. citeturn18view0 |
| Hybrid retrieval (metadata filter → BM25/lexical → embeddings → optional rerank) | 4–10 | ~0.2–2s typical once indexed | **Best overall** for both planning + regressions | Hybrid is not theoretical: entity["company","Anthropic","ai research company"] reports large retrieval-failure reductions by combining embedding retrieval with BM25 (and further with reranking). citeturn12view0 |

### Recommendation: what to build in under 4 hours

Build a **two-lane retrieval CLI** that is git-native and does *not* require prebuilt infrastructure:

**Lane A: fast lexical + metadata (default)**
- Use ripgrep (or `git grep`) to search:
  - frontmatter keys (`type`, `dependencies`, `complexity`, etc.)
  - high-signal sections (`## Outcome` once you add it; until then, `## Notes`, verification commands, constraints).
- Return *paths + section snippets*, not whole files, to keep context compact.

Why this works: ripgrep-style search is already competitive on large repos (sub-second on big trees in published benchmarks). Your archive (hundreds of markdown files) is much smaller than the repos used in those benchmarks. citeturn11view0

**Lane B: “semantic mode” (optional flag)**
- In-session, build embeddings from a *small set of target text* (see granularity section) using a local sentence-transformers model, then do cosine similarity in-memory.
- Only index **Outcome + one-line section headers + key constraints/verification**, not the entire file, to keep build times low and results sharp.

Why this is feasible: sentence-transformers documents CPU throughput for common semantic-search models on the order of hundreds of queries per second, which is enough to embed a few thousand chunks in seconds on a typical dev machine. citeturn16view0

### Upgrade path as the archive grows

When you cross “a few thousand chunks” and semantic lookups become a daily habit, the upgrade is not a server—it's a **rebuildable index artifact** committed to git (or generated in CI and cached by your agent runtime):

- **Step up to FAISS** for fast ANN queries while keeping everything local and rebuildable. FAISS is explicitly designed for efficient similarity search and clustering of dense vectors, including very large collections. citeturn21view0
- Or use **ChromaDB’s ephemeral mode** if you want “batteries included” metadata filtering and don’t want to manage vector math yourself. citeturn18view0
- Add hybrid improvements over time:
  - **BM25-style lexical scoring** (for identifiers, error codes, and API names)
  - **dense embeddings** (for semantic similarity)
  - optional reranking once precision matters more than latency (Anthropic reports additional gains from reranking on top of hybrid retrieval). citeturn12view0

## Granularity recommendation

Granularity is the lever that decides whether retrieval feels like “wow, it *found the thing*” or “it dumped five irrelevant walls of text.”

The evidence base from RAG research and long-context studies is consistent on two points:

- **Optimal granularity varies by query type.** The Mix-of-Granularity (MoG) line of work explicitly motivates that fine-grained questions benefit from finer chunks, while broad questions prefer coarser chunks, and proposes dynamic selection across granularities. citeturn9view0
- **Stuffing more context is not a free win.** Long-context models often recall best when relevant info is near the beginning or end of the prompt and degrade when the answer is buried in the middle. citeturn17view0 This directly argues against “proactively inject a lot of archive text” as your archive grows.

### Best practice for your archive: hierarchical, variable granularity

Use **three layers**, but don’t index them equally:

**Epic level (coarse):** *routing + planning*
- Index: `_epic.md` goals, acceptance criteria, and (ideally) an epic-level “Outcome/Decisions” summary if you add it later.
- Use case fit:
  - **New epic planning:** excellent for “what initiative touched Redis caching / auth / build pipeline?”
  - **PR regression detection:** weak by itself; it’s too high-level.

**Ticket level (medium):** *the default retrieval unit*
- Index: ticket title + frontmatter + Outcome + a small set of canonical fields (constraints, acceptance criteria, verification commands).
- Use case fit:
  - **Planning:** “show me the 3–5 closest tickets and how they were verified.”
  - **Regression checks:** “what invariants did we establish in prior work that this PR might violate?”

**Field/section level (fine):** *precision retrieval, but return the parent ticket*
- Index: small atomic snippets like:
  - constraint bullets
  - invariant statements
  - verification commands
  - key design decisions
- But **return the full ticket path + the surrounding section**, because fine chunks alone can lose context. This mirrors the broader RAG lesson that chunking can “destroy context,” and motivates techniques like contextualizing chunks to improve retrieval. citeturn12view0

image_group{"layout":"carousel","aspect_ratio":"16:9","query":["hybrid RAG BM25 embeddings diagram","document chunking hierarchy retrieval diagram","retrieval augmented generation pipeline diagram"],"num_per_query":1}

### Concrete examples of “useful retrieval” by use case

**PR regression detection (best: ticket + field level)**
- Query input: diff summary + filenames changed + failing test output.
- Retrieval pattern:
  1. **Field-level lexical first:** search for exact identifiers (endpoint path, error code, feature flag name) because BM25/lexical is strong on unique technical strings. citeturn12view0
  2. **Ticket-level semantic second:** find “similar ticket Outcomes” even if identifiers differ (e.g., “added caching with invalidation”).
  3. Return *only* the relevant fields:
     - invariants (“cache must be bypassed for admin role”)
     - verification (“run X; expect Y”)
     - risk notes (“do not change header order due to proxy”)

**New epic planning (best: epic + ticket level, then drill to fields)**
- Query input: short epic pitch + impacted subsystems.
- Retrieval pattern:
  1. Epic-level similarity to find adjacent initiatives (broad recall).
  2. Ticket-level within top epics to find concrete implementation approaches and pitfalls.
  3. Field-level drilldown only when you need “the exact constraint we learned.”

This variable-granularity approach is aligned with MoG’s central premise: broad vs precise questions want different chunking. citeturn9view0

### Proactive injection vs on-demand retrieval

Do **bounded proactive injection** plus **on-demand deepening**. That’s not hand-wavy; it matches documented practice in agent design guidance:

- entity["company","Anthropic","ai research company"] explicitly recommends “just in time” context strategies where agents keep lightweight references (paths, identifiers) and load detailed context via tools at runtime, and also notes that hybrid strategies can retrieve some data up front and explore further as needed. citeturn13view0
- Claude Projects already reflects this trade: when project knowledge approaches the context window limit, it “automatically enable[s] RAG mode” rather than forcing everything into the prompt. citeturn22view0

**When does proactive injection become impractical?**  
If “proactive injection” means “dump lots of raw archive text into every new epic,” it becomes counterproductive well before you reach hundreds of epics, because (a) context is finite, and (b) long-context performance degrades when the relevant bit is buried. citeturn17view0turn13view0

If “proactive injection” instead means “always inject top-K *summaries*,” K can stay small (3–7) even as the archive grows to hundreds of epics—*as long as the injected artifacts are short and high-signal*. That’s exactly what the Outcome proposal is for.

## Cross-session persistence patterns

Across agentic systems, the stable pattern is: **persist structured artifacts to a store, then retrieve and inject them into context when needed.** The differences are mostly (a) where they store, and (b) how automatic the retrieval is.

### What production-facing systems actually do

**Claude Code: file-based persistent instructions + auto-written memory**
- Claude Code uses two cross-session mechanisms: human-written instruction files and auto-written memory notes; both are loaded at session start (with explicit size limits for auto memory). citeturn23view0
- This is directly analogous to your git-native archive: human/agent-authored “Outcome” sections act like durable notes that are easy to retrieve.

**Claude Projects: a knowledge base with automatic RAG when it gets big**
- Projects let you upload documents as a project knowledge base used across chats, and when the knowledge approaches the context window limit, Claude enables RAG mode. citeturn22view0  
This is a strong real-world endorsement of “store artifacts, retrieve selectively,” not “stuff everything into the prompt.”

**Task Master AI: persisted task artifacts in repo files**
- Documentation describes tasks stored in `tasks.json` (and optionally individual task files) with structured fields; it also supports user-defined metadata. citeturn29view0
- Its command reference describes configuration and state stored in repo-local files like `.taskmaster/config.json` and `.taskmaster/state.json`. citeturn14view0  
This is effectively “session handoff via files,” which is the same design center as your `.tickets/_archive/`.

**Shrimp Task Manager: persistent memory + task history backups**
- The project positions itself around persistent memory (“tasks and progress persist across sessions”) and explicitly calls out task-memory backup/restoration. citeturn15view0

**LangGraph: checkpointed state**
- LangGraph describes a persistence layer that saves graph state as checkpoints via a checkpointer, enabling threads to resume with memory. citeturn27view0  
This is more “workflow-engine memory” than “ticket archive memory,” but it reinforces the general mechanism: serialize state, reload later.

**AutoGen: a memory protocol that retrieves and updates context**
- AutoGen documents a Memory protocol intended for retrieving relevant information “just before a specific step” and updating the agent context accordingly, and frames chunking + retrieval quality as the key quality drivers. citeturn28view0

**CrewAI: unified memory with automatic extraction + recall**
- CrewAI documents a “unified memory” that uses composite scoring and can automatically extract discrete facts from task outputs, store them, and recall/inject before subsequent tasks. citeturn26view0  
Even if you don’t adopt CrewAI, the *design lesson* maps cleanly: extract compact facts at task completion → retrieve them later.

### Simplest approach that demonstrably works for your constraints

The simplest proven pattern is: **write durable, compact notes into files, and retrieve them just-in-time.** This is explicitly the recipe in Claude Code’s file-based memory approach and in Anthropic’s broader “just in time” context guidance. citeturn23view0turn13view0

For your git-native archive, that translates to:

- Make each closed ticket produce a **high-signal Outcome block** (below).
- Provide a **single retrieval tool** capable of:
  - lexical search
  - metadata filtering
  - optional semantic search over Outcomes
- Default to **on-demand retrieval**, with a small “proactive top-3” injection for new epic kickoff.

## Structured outcome section proposal

Your current ticket sections (requirements, constraints, acceptance, verification, notes) are *necessary for execution*, but they’re not optimized for *future retrieval*. The missing piece is a closure artifact that answers: **what changed, why, how to verify, and what to watch out for next time**—in a small, consistent shape.

This proposal borrows the high-leverage parts of:
- ADRs: *context → decision → consequences*, with an emphasis on brevity and keeping records in the repo. citeturn24view0turn10search0
- SRE postmortems: *impact, actions taken, root cause, follow-ups*, and the idea that archives become institutional learning when stored and shared. citeturn25view0
- File-based agent memory patterns: tight, high-signal notes loaded into future sessions with explicit size constraints. citeturn23view0turn13view0

### `## Outcome` schema

Target length: **120–200 words**, plus **up to 8 short bullets**. The goal is “small enough that agents don’t skip it; dense enough that retrieval loves it.”

**Template**

```markdown
## Outcome

**Summary:** <2–3 sentences: what shipped/fixed + user-visible effect + scope>

**Key decisions:** 
- <decision> — <why this option; 1 clause>
- <decision> — <tradeoff / alternative rejected>

**Constraints & invariants discovered (keep):**
- <invariant/constraint phrased as a rule>
- <invariant/constraint phrased as a rule>

**Implementation notes (high signal only):**
- Touch points: <paths/modules/apis>
- Pattern: <name the pattern/mechanism used>

**Verification:** 
- <command> → <expected signal>
- <command> → <expected signal>

**Risk / regression surface:** <1–2 bullets max>
- <what might break; what guards it>

**Retrieval tags:** <5–10 keywords and identifiers; include “weird strings” like error codes>
```

Why this improves retrieval quality:
- It creates a **single, dense retrieval target** that *already contains the “contextualization” you otherwise need to prepend at indexing time* (Anthropic’s contextual retrieval work is essentially about restoring lost context around chunks). citeturn12view0
- It makes lexical search dramatically more useful because the “right weird strings” (flags, endpoints, error codes) live in one predictable place.
- It makes semantic search dramatically more reliable because embeddings computed over ~150 words of distilled meaning tend to be less noisy than embeddings over long implementation notes.

### Filled example

Plausible ticket: `type: feature`, adding a lightweight cache with correctness constraints.

```markdown
## Outcome

**Summary:** Added request-scoped + short-TTL caching for `GET /reports/summary` to cut p95 latency under load. Cache is bypassed for privileged/admin views and never caches error responses.

**Key decisions:**
- Use explicit cache key versioning (`reports_summary:v2`) — avoids silent mismatch when response shape changes.
- Cache only successful (200) responses — prevents “sticky” failure modes during partial outages.

**Constraints & invariants discovered (keep):**
- Never cache responses that depend on user role/permissions unless the role is part of the key.
- Cache TTL must remain ≤60s until we have invalidation hooks from the write-path.

**Implementation notes (high signal only):**
- Touch points: `src/reports/summary.ts`, `src/cache/client.ts`, middleware ordering in `src/http/router.ts`
- Pattern: “read-through cache with safe bypass”

**Verification:**
- `pnpm test reports -- --filter summary` → all green
- `curl -H 'X-Role: admin' .../reports/summary` twice → second request must NOT be `X-Cache: HIT`
- `curl .../reports/summary` twice → second request returns `X-Cache: HIT`

**Risk / regression surface:**
- Middleware order matters: auth must run before cache key construction.

**Retrieval tags:** reports, summary, cache, TTL=60, read-through, X-Cache, admin bypass, key version v2
```

This is intentionally “inverted pyramid” writing: the highest-value information is front-loaded, matching ADR guidance on brevity and putting key material first. citeturn24view0