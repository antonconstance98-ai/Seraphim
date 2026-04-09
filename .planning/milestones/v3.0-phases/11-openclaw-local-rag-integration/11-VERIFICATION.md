---
phase: 11-openclaw-local-rag-integration
verified: 2026-04-08T00:00:00Z
status: gaps_found
score: 3/4 success criteria verified
gaps:
  - truth: "rag-post-phase.js fires automatically after Write/Edit to keep the index current"
    status: failed
    reason: >
      rag-post-phase.js exists and is substantive, but it is only registered in the plugin's
      internal hooks/hooks.json file. That file is not read by Claude Code at runtime — it is
      a documentation artifact inside the plugin directory. The hook is absent from
      ~/.claude/settings.json (the only file Claude Code reads for global hooks) and from
      the project's .claude/settings.json. The hook will never fire.
    artifacts:
      - path: "~/.claude/plugins/seraphim/hooks/rag-post-phase.js"
        issue: "Exists and is correct code, but not registered where Claude Code can load it"
      - path: "~/.claude/plugins/seraphim/hooks/hooks.json"
        issue: >
          Internal plugin hooks.json is not a runtime-loaded file. Claude Code does not read
          plugin-local hooks.json files — hooks must appear in ~/.claude/settings.json or
          the project's .claude/settings.json under the hooks key.
    missing:
      - >
        Add a PostToolUse entry for rag-post-phase.js to ~/.claude/settings.json hooks,
        filtered to Write and Edit tool calls, matching the same pattern as the existing
        token-logger and codex-token-logger entries.
human_verification:
  - test: "Run /seraphim:reindex on a project with .seraphim/config.json and verify INDEX_OK output"
    expected: "Command reports index size (e.g., 'INDEX_OK: 48K') and no error"
    why_human: "Requires a live Seraphim project with indexable .seraphim/ artifacts and npm deps installed"
  - test: "Run /seraphim:run on a phase and observe whether '## Prior Project Knowledge' block appears in the phase prompt"
    expected: "After the first reindex, RAG context is injected when the query returns relevant chunks"
    why_human: "Requires a running pipeline session with a populated index"
---

# Phase 11: OpenClaw Local RAG Integration — Verification Report

**Phase Goal:** Research OpenClaw's architecture and adapt the pattern so Seraphim can reference
project history, decisions, and documentation during pipeline phases without external dependencies.

**Verified:** 2026-04-08
**Status:** gaps_found
**Re-verification:** No — initial verification

---

## Goal Achievement

### Observable Truths

| # | Truth | Status | Evidence |
|---|-------|--------|----------|
| 1 | Research document analyzing OpenClaw's RAG architecture exists | VERIFIED | 11-RESEARCH.md is 50+ lines with substantive architecture analysis, technology stack comparison, and recommendation rationale |
| 2 | A local RAG system indexes .seraphim/ artifacts into a searchable SQLite store without cloud services | VERIFIED | lib/rag-indexer.js (606 lines): SQLite via better-sqlite3, FTS5 keyword search, sqlite-vec for vector similarity, @huggingface/transformers for local embeddings; all deps installed in node_modules |
| 3 | Pipeline phases can query project knowledge during execution | VERIFIED | run.md Step 6b-RAG calls query-knowledge.js before each phase dispatch; tools/query-knowledge.js (73 lines) wraps rag-indexer.queryKnowledge with correct --project/--query CLI; wired correctly |
| 4 | Auto re-index hook fires after Write/Edit to keep index current | FAILED | rag-post-phase.js exists and is correct, but is not registered in ~/.claude/settings.json — Claude Code will never invoke it |

**Score:** 3/4 truths verified

---

### Required Artifacts

| Artifact | Expected | Status | Details |
|----------|----------|--------|---------|
| `~/.claude/plugins/seraphim/lib/rag-indexer.js` | Core indexer and query library | VERIFIED | 606 lines, substantive; exports createDb, indexProject, queryKnowledge, isStale; CLI entry point present |
| `~/.claude/plugins/seraphim/tools/query-knowledge.js` | CLI wrapper for pipeline phases | VERIFIED | 73 lines, substantive; correct arg parsing, loads rag-indexer, exits 0 on all error paths per D-09 |
| `~/.claude/plugins/seraphim/hooks/rag-post-phase.js` | PostToolUse hook for auto-reindex | ORPHANED | 60 lines, substantive code, correct fire-and-forget spawn pattern; NOT wired into Claude Code hooks |
| `~/.claude/plugins/seraphim/commands/reindex.md` | /seraphim:reindex command | VERIFIED | 107 lines; full --full flag support, error guidance, incremental vs full rebuild logic |
| `~/.claude/plugins/seraphim/commands/run.md` | Pipeline with RAG injection at Step 6b-RAG | VERIFIED | RAG step confirmed present at lines 138-182; reads rag config, constructs phase query, calls query-knowledge.js, injects ## Prior Project Knowledge block |
| `~/.claude/plugins/seraphim/hooks/hooks.json` | Hook registration | FAILED | File exists but Claude Code does not load plugin-local hooks.json files; this is not a valid registration mechanism |
| `.planning/phases/11-openclaw-local-rag-integration/11-RESEARCH.md` | Research document | VERIFIED | Exists, substantive, covers OpenClaw architecture, SQLite+FTS5+sqlite-vec pattern, embedding model selection, integration rationale |

---

### Key Link Verification

| From | To | Via | Status | Details |
|------|----|-----|--------|---------|
| run.md Step 6b-RAG | tools/query-knowledge.js | bash subprocess call | WIRED | run.md calls `node "$PLUGIN_ROOT/tools/query-knowledge.js" --project ... --query ...` |
| query-knowledge.js | lib/rag-indexer.js | require('../lib/rag-indexer') | WIRED | Line 63: `const { queryKnowledge } = require('../lib/rag-indexer')` |
| rag-post-phase.js | lib/rag-indexer.js | spawned subprocess | WIRED (internally) | Spawns `node rag-indexer.js --reindex` — code is correct |
| rag-post-phase.js | Claude Code PostToolUse event | ~/.claude/settings.json | NOT WIRED | Hook not present in global settings.json; plugin's internal hooks.json is not loaded by Claude Code |
| run.md | lib/config.js rag defaults | node -e inline | WIRED | Step 6b-RAG reads `cfg.rag.enabled` via config.read() |

---

### Data-Flow Trace (Level 4)

| Artifact | Data Variable | Source | Produces Real Data | Status |
|----------|---------------|--------|-------------------|--------|
| run.md RAG_CONTEXT | RAG_CONTEXT shell var | query-knowledge.js stdout | Yes — when index exists and query scores above threshold | FLOWING (conditional on index existing) |
| lib/rag-indexer.js queryKnowledge | context string | SQLite chunks table via FTS5 + cosine similarity | Yes — real DB query with merge/rank logic | FLOWING |

---

### Behavioral Spot-Checks

| Behavior | Command | Result | Status |
|----------|---------|--------|--------|
| rag-indexer.js module loads without error | `node -e "require('./lib/rag-indexer')"` in plugin dir | Exit 0, no output | PASS |
| query-knowledge.js exits 0 on missing index | implied by code path (skip guard at line 56-58) | Code verified; runtime test skipped | SKIP — needs live project |
| @huggingface/transformers installed | `ls node_modules/@huggingface/transformers` | dist/, package.json present | PASS |
| better-sqlite3 installed | `ls node_modules/better-sqlite3` | present | PASS |
| sqlite-vec installed | `ls node_modules/sqlite-vec` | present | PASS |

---

### Anti-Patterns Found

| File | Line | Pattern | Severity | Impact |
|------|------|---------|----------|--------|
| hooks/hooks.json | 19 | `"command": "node $HOME/..."` — registered in plugin-local file only | Blocker | rag-post-phase.js will never fire; index stays stale after Write/Edit |

---

### Human Verification Required

#### 1. reindex command produces a valid index

**Test:** In a project with `.seraphim/config.json` and at least one `.seraphim/phases/*/vision.md`, run `/seraphim:reindex` in Claude Code.
**Expected:** Reports `INDEX_OK: <size>`, no errors. SQLite file appears at `.seraphim/rag/knowledge.sqlite`.
**Why human:** Requires live project and HuggingFace model cache download on first run.

#### 2. RAG context injection in pipeline

**Test:** After indexing, run `/seraphim:run <phase-id> --from discover`. Check if the Discover phase receives a `## Prior Project Knowledge` header in its prompt when relevant chunks exist.
**Expected:** Context block appears; is non-empty; does not break the phase if empty.
**Why human:** Requires a running pipeline session with populated index and relevant artifacts.

---

### Gaps Summary

One gap blocks full goal achievement: the `rag-post-phase.js` hook is correct and complete as a file but is not wired into Claude Code's hook execution path. The plugin's internal `hooks/hooks.json` at `~/.claude/plugins/seraphim/hooks/hooks.json` is not a mechanism that Claude Code reads — hooks must be registered in `~/.claude/settings.json` (global) or the project's `.claude/settings.json` under the `hooks` key.

All other success criteria are met: the research document exists and is substantive, the indexer and query library are implemented with real SQLite/FTS5/vector logic and all npm dependencies installed, and the pipeline's `run.md` has the RAG injection step correctly wired to `query-knowledge.js`.

The gap is isolated and the fix is a single addition to `~/.claude/settings.json` — add a PostToolUse entry pointing to `rag-post-phase.js` filtered to Write and Edit, matching the existing hook pattern in that file.

---

_Verified: 2026-04-08_
_Verifier: Claude (gsd-verifier)_
