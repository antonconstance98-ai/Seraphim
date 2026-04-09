---
phase: 02-progress-visibility
plan: 02
subsystem: seraphim-commands
tags: [dependency-check, rag-indexing, task-types, skills, research]
requires: [01-core-pm-primitives]
provides: [dependency-warning, research-rag-trigger, skills-domain-log]
affects: [start.md, done.md, add-feature.md]
tech-stack:
  added: []
  patterns: [warn-not-block, fire-and-forget-async, try-catch-non-blocking]
key-files:
  created: []
  modified:
    - ~/.claude/plugins/seraphim/commands/start.md
    - ~/.claude/plugins/seraphim/commands/done.md
decisions:
  - "Used indexProject (not indexFile) for RAG — indexFile does not exist in rag-indexer.js exports"
  - "add-feature.md already had depends_on:[] default — no change required"
  - "RAG indexProject called as fire-and-forget Promise (async) to remain non-blocking"
metrics:
  duration: ~5min
  completed: 2026-04-09T18:02:28Z
  tasks_completed: 2
  tasks_total: 2
  files_modified: 2
---

# Phase 02 Plan 02: Dependency Guards and Task Type Extensions Summary

Feature dependency warnings in /seraphim:start plus research RAG indexing and skills domain logging in /seraphim:done.

## Tasks Completed

| Task | Name | Commit | Files |
|------|------|--------|-------|
| 1 | Add dependency check to start.md | 21d4e14 | commands/start.md |
| 2 | Extend done.md for skills domain and research RAG indexing | 0eaf5c8 | commands/done.md |

## What Was Built

**Task 1 — Dependency check in start.md:**
After the WIP limit check and before setting `feature.status = 'in-progress'`, a block filters incomplete dependencies using `findFeature`. If any are incomplete, it emits `DEPS_WARN:dep1 (name) | dep2 (name)` to stdout. The output parsing section handles this with a visible warning banner but continues starting the feature (D-04: warn, not block).

`add-feature.md` already had `depends_on: []` in the feature object literal — no change needed.

**Task 2 — Skills domain log and research RAG indexing in done.md:**
After marking a task complete, if `type === 'skills'` and `foundTask.domain` is set, logs "Skills task completed: {domain}". If `type === 'research'` and `foundTask.notes_path` is set, calls `rag.indexProject(projectRoot)` as a fire-and-forget async call, emits `RAG_INDEXED:` or `RAG_SKIP:` for observability. The entire block is wrapped in try/catch — RAG failure never blocks task completion.

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 1 - Bug] indexFile does not exist in rag-indexer.js**
- **Found during:** Task 2
- **Issue:** The plan spec used `rag.indexFile(task.notes_path, ...)` but rag-indexer.js exports only `indexProject`, `queryKnowledge`, `createDb`, and `isStale`.
- **Fix:** Used `rag.indexProject(projectRoot, {})` as a fire-and-forget async call. This re-indexes the full project including the new notes file, which is the correct behavior anyway.
- **Files modified:** commands/done.md
- **Commit:** 0eaf5c8

## Known Stubs

None. All wired data paths use real functions.

## Self-Check: PASSED

- commands/start.md modified: confirmed (commit 21d4e14)
- commands/done.md modified: confirmed (commit 0eaf5c8)
- DEPS_WARN in start.md: confirmed
- depends_on in add-feature.md: confirmed
- rag-indexer and RAG_INDEXED in done.md: confirmed
