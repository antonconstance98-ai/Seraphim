---
phase: 11-openclaw-local-rag-integration
plan: "01"
subsystem: rag
tags: [rag, sqlite, embeddings, hybrid-search, better-sqlite3, sqlite-vec, huggingface]
dependency_graph:
  requires: []
  provides: [rag-indexer-lib, knowledge-sqlite-schema, hybrid-search]
  affects: [11-02, 11-03]
tech_stack:
  added: [better-sqlite3@12.8.0, sqlite-vec@0.1.9, "@huggingface/transformers@4.0.1"]
  patterns: [singleton-embedder, four-table-sqlite-schema, hybrid-vector-fts-search, mtime-hash-change-detection, js-cosine-fallback]
key_files:
  created:
    - ~/.claude/plugins/seraphim/lib/rag-indexer.js
    - ~/.claude/plugins/seraphim/tests/test-rag-indexer.js
    - ~/.claude/plugins/seraphim/tests/test-rag-fallback.js
    - ~/.claude/plugins/seraphim/package.json
    - ~/.claude/plugins/seraphim/.gitignore
  modified: []
decisions:
  - "package.json created at plugin root (was absent) — no existing npm manifest; Rule 3 auto-fix"
  - "sqlite-vec vec0 table creation wrapped in secondary try/catch in case load() succeeds but vec0 DDL fails — defence in depth"
  - "globSync implemented as recursive sync walker (no external glob dep) — keeps library zero-extra-dep beyond the three declared"
  - "embed() uses singleton pattern with lazy init — avoids re-loading 80MB model on every query call"
metrics:
  duration_minutes: 12
  completed_date: "2026-04-09"
  tasks_completed: 2
  files_created: 5
  files_modified: 0
---

# Phase 11 Plan 01: RAG Core Library Summary

**One-liner:** SQLite-backed hybrid RAG library using all-MiniLM-L6-v2 embeddings with vec0 vector search and graceful JS cosine fallback when sqlite-vec is unavailable.

## What Was Built

`rag-indexer.js` is a CommonJS module that gives any Seraphim project a local knowledge base backed by a four-table SQLite schema. It exports four functions:

- **createDb(dbPath)** — opens or creates the SQLite database, attempts to load the sqlite-vec extension, creates `files`, `chunks`, `chunks_fts`, and (when vec available) `chunks_vec` tables.
- **isStale(db, filePath)** — returns true if a file has never been indexed, or if its mtime+SHA-256 hash has changed since last index.
- **indexProject(projectRoot, options)** — scans Seraphim phase files, planning context files, and source trees; skips unchanged files; chunks, embeds, and stores each chunk; updates the files manifest.
- **queryKnowledge(projectRoot, queryText, options)** — embeds the query, runs hybrid search (vec0 cosine + FTS5 keyword), merges by weighted score (0.7 semantic + 0.3 keyword), applies token budget, returns `{ context, chunks }`.

Two test files provide coverage:
- `test-rag-indexer.js` exercises the full index-then-query path against a real temp directory.
- `test-rag-fallback.js` monkey-patches sqlite-vec to confirm `_vecEnabled === false` and the JS cosine path activates.

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 3 - Blocking] package.json absent from plugin root**
- **Found during:** Task 1 (npm install step)
- **Issue:** The plugin directory had no root-level package.json; `npm install` had no target manifest.
- **Fix:** Created `package.json` with the three required dependencies declared. Also added `.gitignore` to exclude `node_modules/`.
- **Files modified:** `package.json`, `.gitignore`
- **Commit:** 61700cf

**2. [Rule 2 - Missing critical functionality] Secondary try/catch on vec0 DDL**
- **Found during:** Task 1 implementation
- **Issue:** `sqliteVec.load(db)` can succeed (extension loads) but `CREATE VIRTUAL TABLE ... USING vec0` can still throw on some system sqlite builds where the virtual table module is incomplete.
- **Fix:** Wrapped the `CREATE VIRTUAL TABLE chunks_vec` statement in its own try/catch; on failure demotes `db._vecEnabled = false` and logs to stderr.
- **Files modified:** `lib/rag-indexer.js`
- **Commit:** 61700cf

**3. [Rule 2 - Missing critical functionality] globSync as internal recursive walker**
- **Found during:** Task 1 implementation
- **Issue:** Plan referenced glob patterns but no glob dependency was declared and no glob package exists at plugin root. Using external `glob` or `fast-glob` would add an undeclared dependency.
- **Fix:** Implemented `globSync()` as a synchronous recursive directory walker inside `rag-indexer.js`. Handles `**`, `*`, `{ext,ext}` patterns, and exclusion list. Zero extra dependencies.
- **Files modified:** `lib/rag-indexer.js`
- **Commit:** 61700cf

## Known Stubs

None — all exported functions are fully implemented. The embedder downloads `Xenova/all-MiniLM-L6-v2` on first call (~23MB ONNX model cached to `~/.cache/huggingface/`); subsequent calls use the cached model.

## Self-Check: PASSED

Files exist:
- ~/.claude/plugins/seraphim/lib/rag-indexer.js — FOUND
- ~/.claude/plugins/seraphim/tests/test-rag-indexer.js — FOUND
- ~/.claude/plugins/seraphim/tests/test-rag-fallback.js — FOUND

Plugin repo commits:
- 61700cf (Task 1 — feat)
- cc59eb6 (Task 2 — test)

Test results:
- test-rag-fallback.js → PASS
- test-rag-indexer.js → PASS
