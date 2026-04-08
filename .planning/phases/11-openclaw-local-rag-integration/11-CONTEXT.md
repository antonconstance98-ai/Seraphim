# Phase 11: OpenClaw Local RAG Integration - Context

**Gathered:** 2026-04-08
**Status:** Ready for planning

<domain>
## Phase Boundary

Research OpenClaw's architecture (focus on local RAG system) and adapt the pattern so Seraphim pipeline phases can reference project history, decisions, and code during execution. Fully local — no cloud APIs for RAG. Best-quality approach, GPU constraints are not a concern.

</domain>

<decisions>
## Implementation Decisions

### RAG Architecture
- **D-01:** Prioritize the most accurate and efficient approach for best quality. Do not constrain by GPU availability — the RTX 3090 is available and future hardware upgrades are expected.
- **D-02:** Cloud-free requirement: the RAG system must work entirely offline with no external API dependencies. Local embeddings, local vector store, local retrieval.
- **D-03:** Specific technology choices (local embeddings model, vector DB, chunking strategy) are deferred to research. Research should evaluate options and pick the best-quality approach.

### What Gets Indexed
- **D-04:** Three primary artifact categories: (1) pipeline outputs — vision.md, judgment.md, blueprint.md, forge-log.md, crucible.md, (2) decision history — decisions.jsonl, CONTEXT.md files, discussion logs, (3) source code — actual project code files.
- **D-05:** Research should determine the best indexing hygiene — what to include, what to exclude, update frequency, chunking strategy. User is open to research findings here.
- **D-06:** Index must stay current — re-index or incrementally update as new phase outputs and code changes are produced.

### OpenClaw Research Scope
- **D-07:** Architecture overview level — understand OpenClaw's RAG approach conceptually and what patterns are transferable. Not a full reverse-engineering deep dive.
- **D-08:** Compare OpenClaw's approach with other local RAG systems (Cursor, Continue.dev, Aider) at a high level to validate the chosen pattern.

### Query Integration
- **D-09:** Both automatic context injection AND explicit tool calls. Automatic: before each phase runs, the system queries RAG for relevant context and injects it into the phase prompt (phases don't know RAG exists — they just get better context). Explicit: phases can also call a `query_knowledge` tool for specific lookups when they need targeted information.
- **D-10:** Automatic injection should be relevance-filtered — don't inject noise. Only inject context that scores above a relevance threshold for the current phase's task.

### Claude's Discretion
- Specific embedding model choice (sentence-transformers variant, dimension size)
- Vector database selection (ChromaDB, LanceDB, Qdrant, etc.)
- Chunking strategy (fixed-size, semantic, document-level)
- Relevance threshold for automatic injection
- Index update trigger (post-phase hook, periodic, on-demand)
- How much context to inject per phase (token budget for RAG context)

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### OpenClaw
- OpenClaw GitHub repository (to be cloned during research) — focus on RAG/indexing architecture, not full codebase

### Design Spec
- `docs/specs/2026-04-04-seraphim-v3-design.md` — Phase definitions, `.seraphim/` directory structure, phase output file conventions

### Prior Phase Context
- `.planning/phases/03-six-phase-pipeline-and-profile-management/03-CONTEXT.md` — Pipeline structure, phase output schemas
- `.planning/phases/07-multi-project-dashboard/07-CONTEXT.md` — Data sources per project (D-07: config, token-log, decisions, phase state, outputs)

### Existing Tools
- `.planning/phases/02-model-executors-and-pricing/02-CONTEXT.md` — Executor interface pattern (relevant for how RAG tool call would be exposed)

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- No existing RAG infrastructure — this is greenfield
- fetchdocs.js — Context7 endpoint pattern (could inform external doc retrieval if needed)
- Qwen executor already runs local inference via ollama — pattern for local GPU model usage

### Established Patterns
- `.seraphim/` directory holds all per-project state — RAG index would live here too
- JSONL append-only logging — decisions.jsonl is a primary indexing target
- Phase output files are markdown — straightforward to chunk and embed

### Integration Points
- RAG context injection hooks into the pipeline orchestrator (Phase 3, plan 03-06) before each phase prompt is assembled
- Explicit query_knowledge tool would be available to phase command prompts
- Index updates triggered by phase completion (same hook point as dashboard data push)

</code_context>

<specifics>
## Specific Ideas

- User wants to take inspiration specifically from OpenClaw's design — how it creates local RAG systems for project knowledge referencing. This is the primary research target.
- The RAG system should feel invisible to phases during normal operation (automatic injection) but be queryable when a phase needs specific information (explicit tool).

</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope

</deferred>

---

*Phase: 11-openclaw-local-rag-integration*
*Context gathered: 2026-04-08*
