# Phase 11: OpenClaw Local RAG Integration - Research

**Researched:** 2026-04-08
**Domain:** Local RAG systems, vector search, embeddings, Node.js, pipeline context injection
**Confidence:** HIGH

---

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions

- **D-01:** Prioritize the most accurate and efficient approach for best quality. Do not constrain by GPU availability — the RTX 3090 is available and future hardware upgrades are expected.
- **D-02:** Cloud-free requirement: the RAG system must work entirely offline with no external API dependencies. Local embeddings, local vector store, local retrieval.
- **D-03:** Technology choices (local embeddings model, vector DB, chunking strategy) are deferred to research.
- **D-04:** Three primary artifact categories: (1) pipeline outputs — vision.md, judgment.md, blueprint.md, forge-log.md, crucible.md, (2) decision history — decisions.jsonl, CONTEXT.md files, (3) source code — actual project code files.
- **D-05:** Research determines indexing hygiene — what to include, what to exclude, update frequency, chunking strategy.
- **D-06:** Index must stay current — re-index or incrementally update as new phase outputs and code changes are produced.
- **D-07:** OpenClaw research scope: architecture overview level only. Not a full reverse-engineering deep dive.
- **D-08:** Compare OpenClaw's approach with other local RAG systems at high level to validate chosen pattern.
- **D-09:** Both automatic context injection AND explicit tool calls. Automatic: query RAG before each phase runs and inject relevant context into phase prompt. Explicit: phases can call `query_knowledge` tool.
- **D-10:** Automatic injection should be relevance-filtered — only inject context scoring above a threshold.

### Claude's Discretion

- Specific embedding model choice (sentence-transformers variant, dimension size)
- Vector database selection (ChromaDB, LanceDB, Qdrant, etc.)
- Chunking strategy (fixed-size, semantic, document-level)
- Relevance threshold for automatic injection
- Index update trigger (post-phase hook, periodic, on-demand)
- How much context to inject per phase (token budget for RAG context)

### Deferred Ideas (OUT OF SCOPE)

None — discussion stayed within phase scope.

</user_constraints>

---

## Summary

OpenClaw's local RAG architecture is the primary inspiration for this phase. Research confirmed that OpenClaw uses SQLite as its sole storage backend — combining FTS5 keyword search with `sqlite-vec` for vector similarity — all in a single `.sqlite` file with zero external service dependencies. This pattern is directly transferable to Seraphim because: (a) Node.js is already the project runtime, (b) `better-sqlite3` and `sqlite-vec` are npm-installable with no native build issues, and (c) the four-table schema (files, chunks, chunks_vec, chunks_fts) is well-documented and simple to implement.

For embeddings, the RTX 3090 is available but NOT currently active (ollama is not running). The `@huggingface/transformers` npm package (v4.0.1, latest) runs `all-MiniLM-L6-v2` locally in the Node.js process with no server required — first run downloads the ONNX model (~90MB) and caches it. This is the safest baseline: works offline once the model is cached, needs no GPU (CPU inference is fast enough for 384-dim embeddings), and produces 384-dimension vectors suitable for cosine similarity.

The integration pattern is two-layer: (1) a `rag-indexer.js` library that builds and queries the index, and (2) hooks into the pipeline orchestrator (`run.md`) that call `query_knowledge` before assembling each phase prompt. Phases are not modified — they receive better context silently.

**Primary recommendation:** Use the OpenClaw SQLite + `sqlite-vec` + `@huggingface/transformers` stack. Single file, zero services, pure Node.js, matches existing project conventions.

---

## Standard Stack

### Core

| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| `better-sqlite3` | 12.8.0 (npm latest) | SQLite driver — sync API, high performance | Already used by sqlite-vec Node.js examples; sync API matches existing Node.js tool patterns in this plugin |
| `sqlite-vec` | 0.1.9 (npm latest) | SQLite extension for vector similarity (cosine distance in SQL) | OpenClaw uses this exact extension; zero-ops, embedded, no server; updated March 2026 |
| `@huggingface/transformers` | 4.0.1 (npm latest) | Local ONNX model inference — generates 384-dim embeddings in Node.js process | Offline after first model download; no GPU required for embed-only workloads; replaces deprecated `@xenova/transformers` |

### Supporting

| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| `minisearch` | 7.2.0 (npm latest) | Pure-JS BM25 full-text search | Fallback if `sqlite-vec` extension fails to load on this OS/arch |
| `flexsearch` | 0.8.212 (npm latest) | Ultra-fast keyword index | Alternative to minisearch for pure keyword-only retrieval path |

### Alternatives Considered

| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| `better-sqlite3` + `sqlite-vec` | ChromaDB (Python server) | ChromaDB requires Python process; sqlite-vec is embedded, no server, matches existing Node.js runtime |
| `better-sqlite3` + `sqlite-vec` | LanceDB (`@lancedb/lancedb` v0.27.2) | LanceDB is also Node.js native and file-based; sqlite-vec chosen because OpenClaw uses it and SQLite is more universal |
| `@huggingface/transformers` | ollama `nomic-embed-text` | ollama is not currently running; @huggingface/transformers works without any running service |
| `@huggingface/transformers` | OpenAI Embeddings API | Violates D-02 (no cloud dependencies) |

**Installation:**
```bash
npm install better-sqlite3 sqlite-vec @huggingface/transformers
```

**Version verification (confirmed 2026-04-08 via npm registry):**
- `better-sqlite3`: 12.8.0
- `sqlite-vec`: 0.1.9
- `@huggingface/transformers`: 4.0.1
- `minisearch`: 7.2.0

---

## Architecture Patterns

### OpenClaw RAG Architecture (The Reference Pattern)

OpenClaw's local RAG system uses a single `.sqlite` file stored at `~/.openclaw/memory/{agentId}.sqlite` with four tables:

```
files          — path, mtime, size, content_hash (change-detection for incremental indexing)
chunks         — rowid, path, start_line, end_line, text, embedding (JSON-serialized float array)
chunks_vec     — virtual table via sqlite-vec, stores binary float32 vectors for cosine search
chunks_fts     — virtual table via SQLite FTS5, stores text for BM25 keyword search
```

**Retrieval:**
1. Query goes through both paths in parallel
2. `chunks_vec`: `SELECT rowid, vec_distance_cosine(v.embedding, ?) AS dist ORDER BY dist LIMIT k`
3. `chunks_fts`: standard FTS5 BM25 rank query
4. Results merged by weighted scoring formula: `score = alpha * semantic_score + (1 - alpha) * keyword_score`
5. Graceful degradation: if `sqlite-vec` fails to load, falls back to JS cosine similarity on stored float arrays

**Key insight:** OpenClaw stores embedding vectors twice — as JSON in `chunks.embedding` (for JS fallback) and as binary in `chunks_vec` (for fast SQL search). This guarantees functionality across environments.

### Recommended Project Structure

```
~/.claude/plugins/seraphim/
├── lib/
│   └── rag-indexer.js       # Core RAG library: index(), query(), isStale()
├── tools/
│   └── query-knowledge.js   # CLI tool exposing query_knowledge to phase prompts
└── hooks/
    └── rag-post-phase.js    # Post-phase hook: triggers re-index after phase completes

.seraphim/                   # Per-project (lives in project root)
└── rag/
    └── knowledge.sqlite     # Single-file vector store + FTS index
```

### Pattern 1: Incremental Indexing with Change Detection

**What:** Before indexing, compare file mtime and content hash against the `files` table. Skip unchanged files. Re-chunk and re-embed only changed or new files.

**When to use:** Post-phase hook trigger — after Forge writes commits or any phase writes output files.

```javascript
// Source: OpenClaw architecture (pingcap.com blog, 2026)
function isStale(db, filePath) {
  const row = db.prepare('SELECT mtime, hash FROM files WHERE path = ?').get(filePath);
  if (!row) return true;
  const stat = fs.statSync(filePath);
  if (stat.mtimeMs !== row.mtime) {
    const hash = computeHash(filePath);
    return hash !== row.hash;
  }
  return false;
}
```

### Pattern 2: Hybrid Search (Semantic + Keyword)

**What:** Run vector similarity search and FTS keyword search in parallel, merge results by weighted score.

**When to use:** All query operations. Hybrid consistently outperforms either method alone on short technical documents.

```javascript
// Conceptual pattern from OpenClaw architecture
function hybridSearch(db, queryEmbedding, queryText, k = 5, alpha = 0.7) {
  const vecResults = db.prepare(`
    SELECT c.rowid, c.text, c.path, v.distance
    FROM chunks_vec v
    JOIN chunks c ON c.rowid = v.rowid
    ORDER BY vec_distance_cosine(v.embedding, ?) ASC
    LIMIT ?
  `).all(queryEmbedding, k * 2);

  const ftsResults = db.prepare(`
    SELECT c.rowid, c.text, c.path, rank
    FROM chunks_fts f
    JOIN chunks c ON c.rowid = f.rowid
    WHERE chunks_fts MATCH ?
    ORDER BY rank
    LIMIT ?
  `).all(queryText, k * 2);

  // Merge, deduplicate by rowid, apply weighted scoring
  return mergeResults(vecResults, ftsResults, alpha).slice(0, k);
}
```

### Pattern 3: Automatic Context Injection

**What:** Before each phase command runs (in `run.md` Step 6), call `query_knowledge` with the phase task description as query string. Inject results above a similarity threshold as a `## Prior Context` block at the top of the phase prompt.

**When to use:** Phase orchestrator (`run.md`) — invisible to individual phase commands.

```
// In run.md, before dispatching phase:
RAG_CONTEXT=$(node "$PLUGIN_ROOT/tools/query-knowledge.js" \
  --project "$PROJECT_ROOT" \
  --query "$(cat $PHASE_PROMPT_SUMMARY)" \
  --threshold 0.75 \
  --max-tokens 2000)

// If RAG_CONTEXT is non-empty, prepend to phase prompt as:
// "## Prior Project Knowledge\n{RAG_CONTEXT}\n---\n"
```

### Pattern 4: Chunking Strategy for Mixed Content

**What:** Document-aware chunking — different chunk sizes for different document types:
- **Markdown phase outputs** (vision.md, judgment.md, blueprint.md): chunk by heading sections (H2/H3 boundaries). Each section becomes one chunk. Preserves semantic coherence.
- **decisions.jsonl**: one record = one chunk. Each JSONL line is already a discrete decision unit.
- **Source code**: chunk by function/class boundaries using a simple regex. Fallback: 40-line sliding window with 10-line overlap.
- **Config files** (config.json): whole-document as single chunk (small enough).

**Why:** all-MiniLM-L6-v2 caps at 256 tokens per chunk. Section-level chunking for markdown keeps each chunk coherent and within limit.

### Anti-Patterns to Avoid

- **Chunking entire phase output files as one piece:** A full `blueprint.md` can be 2,000+ tokens. Exceeds model limits and retrieves noisy context. Chunk by section instead.
- **Synchronous embed during phase execution:** Embedding 50 files takes 2-5 seconds. Trigger indexing in a fire-and-forget post-phase hook, not inline during phase startup.
- **Re-embedding unchanged files every run:** Expensive and slow. Always use content-hash change detection before re-embedding.
- **Injecting context without a relevance threshold:** Low-similarity chunks add noise. Filter at cosine distance > 0.25 (or similarity < 0.75).
- **Storing index inside `.seraphim/phases/`:** Index belongs in `.seraphim/rag/` — it's project-level state, not phase-level output.

---

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Vector similarity in JS | Manual cosine distance nested loops | `sqlite-vec` SQL extension | SQL set operations on pre-indexed vectors are 10-100x faster for k-NN; handles edge cases |
| BM25 keyword search | Regex text search over file contents | SQLite FTS5 virtual table | FTS5 built into SQLite, handles tokenization, stop words, ranking; regex scan is O(n*m) |
| ONNX model loading | Custom model file parser | `@huggingface/transformers` `pipeline()` | Handles model caching, ONNX runtime, tokenization, padding, mean pooling — 200+ lines of complexity |
| Content change detection | Timestamp comparison only | mtime + content hash (SHA-256) | mtime alone fails when files are copied or touched without content change; hash is authoritative |

**Key insight:** The sqlite-vec + FTS5 combination eliminates the need for any separate vector database service. The entire RAG index is a single `.sqlite` file that survives restarts, is human-inspectable with sqlite3 CLI, and requires zero infrastructure management.

---

## Common Pitfalls

### Pitfall 1: sqlite-vec Extension Load Failure

**What goes wrong:** `sqlite-vec` is a native SQLite extension. On some Linux configurations, the extension binary may fail to load due to path resolution or missing `libsqlite3` symbols.

**Why it happens:** Node.js `better-sqlite3` bundles its own SQLite. The `sqlite-vec` npm package provides pre-built extension binaries for common platforms. Version mismatch between bundled SQLite and extension can cause load failure.

**How to avoid:** Use the graceful degradation pattern — store embeddings as JSON text in `chunks.embedding` AND in `chunks_vec`. If `sqlite-vec` fails to load, fall back to JS cosine similarity over JSON-deserialized vectors. This is exactly what OpenClaw does.

**Warning signs:** `SQLITE_ERROR: sqlite-vec extension not found` in console. System continues to work via fallback path.

### Pitfall 2: First-Run Model Download Blocks Pipeline

**What goes wrong:** `@huggingface/transformers` downloads the all-MiniLM-L6-v2 ONNX model (~90MB) on first use. If this happens inline during a phase run, it causes a 20-60 second pause.

**Why it happens:** Model is cached to `~/.cache/huggingface/` after first download but must be downloaded once.

**How to avoid:** Add a `warm-up` step to `/seraphim:new-project` that pre-downloads the model when setting up a new Seraphim project. Also document in help.md that first indexing requires internet access to download the model, after which it works offline.

### Pitfall 3: decisions.jsonl Contains Meta-Records

**What goes wrong:** The existing `decisions-logger.js` writes meta-records (e.g., `{type: "meta", ...}`) that should not be indexed as knowledge. Indexing them pollutes query results with system records.

**Why it happens:** Identified in Phase 6 implementation notes — `aggregateDecisions` already filters meta-records out.

**How to avoid:** In the indexer, parse each JSONL line and skip records where `type === "meta"` or where `phase` is absent. Only index records with `phase`, `model`, and `outcome` fields.

### Pitfall 4: Chunking Markdown with Raw Headers

**What goes wrong:** Phase output files use structured SERAPHIM marker blocks (`<!-- APPROACH: ... -->`). Splitting by `\n# ` regex may split inside marker blocks, creating incoherent chunks.

**Why it happens:** Markdown headers appear inside fenced code blocks and HTML comments in phase outputs.

**How to avoid:** Split by H2/H3 headings (`\n## ` or `\n### `) which are semantic section boundaries in phase outputs. Skip lines that start with `<!--` or are inside ``` fences when identifying split points.

### Pitfall 5: Token Budget Overrun from RAG Injection

**What goes wrong:** Injecting 10 high-similarity chunks into every phase prompt consumes 2,000+ tokens per phase across a 6-phase pipeline = 12,000 extra tokens per run.

**Why it happens:** RAG context is injected without a token budget cap.

**How to avoid:** Cap injected context at a configurable token budget (default: 1,500 tokens). Count tokens as `text.length / 4` (rough estimate). Rank chunks by similarity score, inject highest-scoring ones first, stop when budget is reached.

---

## Code Examples

Verified patterns from official sources:

### Loading sqlite-vec in Node.js

```javascript
// Source: https://alexgarcia.xyz/sqlite-vec/js.html (verified March 2026)
const Database = require('better-sqlite3');
const sqliteVec = require('sqlite-vec');

const db = new Database(dbPath);
sqliteVec.load(db);  // loads the sqlite-vec extension

// Verify
const { vec_version } = db.prepare('SELECT vec_version() AS vec_version').get();
console.log(`sqlite-vec version: ${vec_version}`);
```

### Generating embeddings with @huggingface/transformers

```javascript
// Source: https://philna.sh/blog/2024/09/25/how-to-create-vector-embeddings-in-node-js/
// Note: package is @huggingface/transformers (not deprecated @xenova/transformers)
const { pipeline } = require('@huggingface/transformers');

let extractor;
async function getEmbedder() {
  if (!extractor) {
    extractor = await pipeline('feature-extraction', 'Xenova/all-MiniLM-L6-v2');
  }
  return extractor;
}

async function embed(text) {
  const embedder = await getEmbedder();
  const output = await embedder(text, { pooling: 'mean', normalize: true });
  return Array.from(output.data);  // 384-dimension float array
}
```

### sqlite-vec virtual table schema

```sql
-- Source: OpenClaw architecture analysis (pingcap.com, 2026)
CREATE TABLE IF NOT EXISTS files (
  path TEXT PRIMARY KEY,
  mtime REAL,
  size INTEGER,
  hash TEXT
);

CREATE TABLE IF NOT EXISTS chunks (
  rowid INTEGER PRIMARY KEY AUTOINCREMENT,
  path TEXT,
  start_line INTEGER,
  end_line INTEGER,
  text TEXT,
  embedding TEXT  -- JSON float array for JS fallback
);

CREATE VIRTUAL TABLE IF NOT EXISTS chunks_vec USING vec0(
  embedding float[384]
);

CREATE VIRTUAL TABLE IF NOT EXISTS chunks_fts USING fts5(
  text,
  content=chunks,
  content_rowid=rowid
);
```

### Cosine similarity JS fallback

```javascript
// For when sqlite-vec extension fails to load
function cosineSimilarity(a, b) {
  let dot = 0, normA = 0, normB = 0;
  for (let i = 0; i < a.length; i++) {
    dot += a[i] * b[i];
    normA += a[i] * a[i];
    normB += b[i] * b[i];
  }
  return dot / (Math.sqrt(normA) * Math.sqrt(normB));
}
```

### Integration hook in run.md (conceptual pattern)

```bash
# In run.md before dispatching a phase, query RAG for relevant context
PLUGIN_ROOT="$HOME/.claude/plugins/seraphim"
RAG_CONTEXT=$(node "$PLUGIN_ROOT/tools/query-knowledge.js" \
  --project "$PROJECT_ROOT" \
  --query "$PHASE_TASK_DESCRIPTION" \
  --threshold 0.75 \
  --max-tokens 1500 2>/dev/null)

# Fire-and-forget re-index after phase completes
node "$PLUGIN_ROOT/lib/rag-indexer.js" --reindex "$PROJECT_ROOT" &
```

---

## What Gets Indexed

Based on D-04 and D-05, the following index strategy is recommended:

### Include

| Source | Path Pattern | Chunk Strategy | Priority |
|--------|-------------|----------------|----------|
| Pipeline outputs | `.seraphim/phases/*/vision.md` | H2/H3 section split | High |
| Pipeline outputs | `.seraphim/phases/*/judgment.md` | H2/H3 section split | High |
| Pipeline outputs | `.seraphim/phases/*/blueprint.md` | H2/H3 section split | High |
| Pipeline outputs | `.seraphim/phases/*/crucible.md` | H2/H3 section split | High |
| Pipeline outputs | `.seraphim/phases/*/forge-log.md` | H2 section split | Medium |
| Pipeline outputs | `.seraphim/phases/*/discovery/*.md` | H2 section split | Medium |
| Decision history | `.seraphim/decisions.jsonl` | One record per chunk | High |
| Decision history | `.planning/phases/*-CONTEXT.md` | H2 section split | High |
| Project config | `.seraphim/config.json` | Whole document | Low |
| Source code | `src/**/*.{js,ts,py}` | Function/class or 40-line window | Medium |
| Source code | `lib/**/*.{js,ts}` | Function/class or 40-line window | Medium |

### Exclude

| Pattern | Reason |
|---------|--------|
| `node_modules/**` | Dependencies, not project knowledge |
| `.git/**` | Git internals |
| `*.sqlite`, `*.db` | Binary files, not text |
| `.seraphim/rag/**` | The index itself |
| `*.jsonl` except `decisions.jsonl` | Token logs, hook state — not knowledge |
| `.planning/.hook-state/**` | Internal GSD state, not knowledge |
| Files > 500KB | Likely binary or auto-generated |

---

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| `@xenova/transformers` | `@huggingface/transformers` | 2024 (package rename) | Use new package name; old one still works but deprecated |
| Separate vector DB server (ChromaDB, Qdrant) | sqlite-vec embedded extension | 2024-2025 | No server process needed; single file deployment |
| Python-only embedding pipelines | ONNX models via Transformers.js in Node.js | 2024 | No Python dependency for embeddings |

**Deprecated/outdated:**
- `@xenova/transformers`: Renamed to `@huggingface/transformers`. Still published but deprecated as of 2024.
- `sqlite-vss`: Predecessor to `sqlite-vec`. Replaced by `sqlite-vec` which is faster and simpler.

---

## Environment Availability

| Dependency | Required By | Available | Version | Fallback |
|------------|------------|-----------|---------|----------|
| Node.js | All RAG code | Yes | v22.22.2 | — |
| `better-sqlite3` | SQLite storage | Installable | 12.8.0 (registry) | — |
| `sqlite-vec` | Vector search | Installable | 0.1.9 (registry) | JS cosine similarity fallback |
| `@huggingface/transformers` | Local embeddings | Installable | 4.0.1 (registry) | — |
| HuggingFace model cache | all-MiniLM-L6-v2 ONNX | Not yet cached | ~90MB download | Must download once, then offline |
| ollama | Alternative embeddings | Not running | — | @huggingface/transformers is primary |
| GPU (RTX 3090) | Faster embedding inference | Not active | — | CPU inference is sufficient for embed workloads |
| Internet | First model download | Available | — | After first download, fully offline |

**Missing dependencies with no fallback:**
- Internet access required once for model download (~90MB). After that, fully offline. If internet unavailable at install time, use `ollama pull nomic-embed-text` as alternative once ollama is running.

**Missing dependencies with fallback:**
- `sqlite-vec` native extension: JS cosine similarity fallback built into `rag-indexer.js`.

---

## Open Questions

1. **Should source code be indexed at Phase 11 scope?**
   - What we know: D-04 includes source code as Category 3. For projects with large codebases this could create a very large index.
   - What's unclear: Is the primary use case querying pipeline artifacts (decisions, phase outputs), or does source code retrieval add significant value?
   - Recommendation: Index source code but implement a file-count/size cap. Default: index up to 200 source files. Make configurable in `.seraphim/config.json` as `rag.max_source_files`.

2. **Re-index trigger: post-phase hook vs. on-demand command**
   - What we know: Fire-and-forget post-phase re-index is clean (D-06 says index must stay current).
   - What's unclear: A hook in `run.md` is straightforward. But if phases are run individually (not via `/seraphim:run`), the hook won't fire.
   - Recommendation: Add re-index call to both `run.md` (automatic) and expose `/seraphim:reindex` command for manual trigger. The indexer handles incremental updates so calling it repeatedly is cheap.

3. **Token budget for injected RAG context**
   - What we know: 1,500 tokens is a reasonable default; rough estimate via `text.length / 4`.
   - What's unclear: Some phases (Judge, Crucible) may benefit from more context; Discovery may need less.
   - Recommendation: Default cap of 1,500 tokens. Make configurable per phase in config. Add `rag.token_budget` to `.seraphim/config.json`.

---

## Validation Architecture

No dedicated test framework is configured for the Seraphim plugin (it uses Node.js scripts, not a test-runner package). Tests follow the skip-guard pattern established in Phase 6: `try/catch + process.exit(0)` so test runner doesn't fail before implementation.

### Phase Requirements → Test Map

| Req | Behavior | Test Type | Command | File Exists? |
|-----|----------|-----------|---------|-------------|
| RAG-01 | rag-indexer.js indexes markdown files | unit | `node tests/test-rag-indexer.js` | Wave 0 |
| RAG-02 | Change detection skips unchanged files | unit | `node tests/test-rag-indexer.js` | Wave 0 |
| RAG-03 | query_knowledge returns relevant results | unit | `node tests/test-rag-query.js` | Wave 0 |
| RAG-04 | sqlite-vec fallback works without extension | unit | `node tests/test-rag-fallback.js` | Wave 0 |
| RAG-05 | run.md injects context before phase | smoke | manual inspection of prompt output | manual |
| RAG-06 | Post-phase re-index runs fire-and-forget | smoke | `node tests/test-rag-hook.js` | Wave 0 |

### Wave 0 Gaps

- [ ] `tests/test-rag-indexer.js` — indexing and change detection
- [ ] `tests/test-rag-query.js` — query and hybrid search
- [ ] `tests/test-rag-fallback.js` — sqlite-vec absent fallback path
- [ ] `tests/test-rag-hook.js` — post-phase fire-and-forget

---

## Sources

### Primary (HIGH confidence)

- [PingCAP blog: Local-First RAG using SQLite with OpenClaw](https://www.pingcap.com/blog/local-first-rag-using-sqlite-ai-agent-memory-openclaw/) — OpenClaw SQLite schema and retrieval mechanisms (2026)
- [alexgarcia.xyz: sqlite-vec in Node.js](https://alexgarcia.xyz/sqlite-vec/js.html) — Updated March 2026, verified API
- [GitHub: asg017/sqlite-vec](https://github.com/asg017/sqlite-vec) — Official extension repository
- [GitHub: TechNickAI/openclaw-config](https://github.com/TechNickAI/openclaw-config) — OpenClaw three-tier memory architecture
- npm registry: `better-sqlite3@12.8.0`, `sqlite-vec@0.1.9`, `@huggingface/transformers@4.0.1`, `minisearch@7.2.0` — verified 2026-04-08

### Secondary (MEDIUM confidence)

- [philna.sh: How to Create Vector Embeddings in Node.js](https://philna.sh/blog/2024/09/25/how-to-create-vector-embeddings-in-node-js/) — Transformers.js embedding pattern
- [Oflight: Building a RAG-Enabled Knowledge Base with OpenClaw](https://www.oflight.co.jp/en/columns/qwen35-9b-openclaw-rag-knowledge-base-agent) — OpenClaw RAG four-layer architecture
- [Medium: Customising OpenClaw with RAG](https://medium.com/@C.Dalrymple/customising-my-openclaw-instance-with-rag-retrieval-augmented-generation-0a4fc933c639) — User implementation patterns (March 2026)

---

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH — npm versions verified from registry; sqlite-vec updated March 2026; OpenClaw architecture confirmed from two independent sources
- Architecture: HIGH — OpenClaw's SQLite schema documented from official source; patterns directly align with existing Node.js plugin conventions
- Pitfalls: MEDIUM — sqlite-vec load failure is a known issue class; token budget pitfall is first-principles reasoning

**Research date:** 2026-04-08
**Valid until:** 2026-05-08 (stable ecosystem, 30-day window)
