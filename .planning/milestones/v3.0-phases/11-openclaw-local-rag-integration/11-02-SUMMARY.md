---
phase: 11-openclaw-local-rag-integration
plan: "02"
subsystem: rag
tags: [rag, cli, hooks, query, reindex]
dependency_graph:
  requires: [11-01]
  provides: [query-knowledge-cli, rag-post-phase-hook]
  affects: [hooks/hooks.json, lib/rag-indexer.js]
tech_stack:
  added: []
  patterns: [fire-and-forget-spawn, stdin-stdout-hook, skip-guard-tests]
key_files:
  created:
    - ~/.claude/plugins/seraphim/tools/query-knowledge.js
    - ~/.claude/plugins/seraphim/hooks/rag-post-phase.js
    - ~/.claude/plugins/seraphim/tests/test-rag-query.js
    - ~/.claude/plugins/seraphim/tests/test-rag-hook.js
  modified:
    - ~/.claude/plugins/seraphim/lib/rag-indexer.js
    - ~/.claude/plugins/seraphim/hooks/hooks.json
decisions:
  - "query-knowledge.js exits 0 silently on all error paths — phases must never fail because RAG is unavailable (D-09)"
  - "rag-post-phase.js uses detached spawn + unref() pattern for true fire-and-forget — avoids keeping Node event loop alive"
  - "hooks.json uses command key (not script) for rag-post-phase entry to match full absolute path pattern"
metrics:
  duration: "8 min"
  completed: "2026-04-09"
  tasks_completed: 3
  files_created: 4
  files_modified: 2
---

# Phase 11 Plan 02: RAG Query CLI and Post-Phase Hook Summary

**One-liner:** query-knowledge.js CLI wraps rag-indexer.queryKnowledge for phase context retrieval; rag-post-phase.js hook triggers fire-and-forget reindex after every Write/Edit.

## What Was Built

Three tasks completed building the two consumer surfaces on top of rag-indexer.js from Plan 01.

**Task 1 — query-knowledge.js CLI tool** (`tools/query-knowledge.js`)

Standalone Node.js CLI invoked as `node query-knowledge.js --project <root> --query <text>`. Exits 0 with empty stdout when the index doesn't exist, args are missing, or any error occurs. Prints context string to stdout on success. No external dependencies — uses `process.argv` directly.

**Task 2 — rag-post-phase.js hook + hooks.json registration** (`hooks/rag-post-phase.js`, `hooks/hooks.json`, `lib/rag-indexer.js`)

PostToolUse hook following the same stdin/stdout event pattern as token-logger.js and session-start.js. Walks up from `event.cwd` looking for `.seraphim/config.json` to identify the project root. Spawns a detached background `node rag-indexer.js --reindex <root>` process and calls `.unref()` so the hook exits immediately. Responds with `{ continue: true }` and exits 0 in all cases.

Also added a `--reindex` CLI entry point to rag-indexer.js (`require.main === module` block) so the spawned subprocess can invoke `indexProject` directly.

hooks.json updated with a new PostToolUse entry filtered to Write and Edit tool calls.

**Task 3 — Test scaffolds** (`tests/test-rag-query.js`, `tests/test-rag-hook.js`)

Both use the skip-guard pattern (try/catch + process.exit(0)). test-rag-query asserts empty stdout + exit 0 for nonexistent project. test-rag-hook spawns the hook with synthetic stdin and asserts `{ continue: true }` response within 5 seconds. Both log PASS.

## Verification Results

```
CLI exit: 0                            ✓ query-knowledge exits cleanly
{"continue":true}                      ✓ rag-post-phase responds correctly
rag-post-phase in hooks.json: true     ✓ hook registered
PASS: test-rag-query (no-index case)   ✓
PASS: test-rag-hook                    ✓
```

## Commits

| Task | Commit | Message |
|------|--------|---------|
| 1 | 9010ea9 | feat(11-02): add query-knowledge.js CLI tool |
| 2 | 41a39c9 | feat(11-02): add rag-post-phase hook and wire into hooks.json |
| 3 | 37e3d31 | test(11-02): add smoke tests for query-knowledge CLI and rag-post-phase hook |

## Deviations from Plan

None — plan executed exactly as written.

## Known Stubs

None. Both surfaces are fully wired to rag-indexer.js.

## Self-Check: PASSED

- [x] `~/.claude/plugins/seraphim/tools/query-knowledge.js` exists
- [x] `~/.claude/plugins/seraphim/hooks/rag-post-phase.js` exists
- [x] `~/.claude/plugins/seraphim/tests/test-rag-query.js` exists
- [x] `~/.claude/plugins/seraphim/tests/test-rag-hook.js` exists
- [x] Commits 9010ea9, 41a39c9, 37e3d31 present in plugin repo
- [x] hooks.json contains rag-post-phase registration
- [x] rag-indexer.js has --reindex CLI entry point
