---
phase: 33-core-command-layer
plan: "01"
subsystem: seraphim-lib
tags: [seed-store, requirements, wave-planner, kahn-algorithm, lib]
dependency_graph:
  requires: []
  provides: [seed-store.js, requirements.js, wave-planner.js]
  affects: [33-02, 33-03, 33-04, 33-05]
tech_stack:
  added: []
  patterns: [atomic-tmp-rename, jsonl-append, kahnsBFS, regex-yaml-parse]
key_files:
  created:
    - ~/.claude/plugins/seraphim/lib/seed-store.js
    - ~/.claude/plugins/seraphim/lib/requirements.js
    - ~/.claude/plugins/seraphim/lib/wave-planner.js
  modified: []
decisions:
  - "parseFrontmatter strips surrounding quotes from string values to match expected output (Test Idea, not 'Test Idea')"
  - "YAML trigger_conditions stored with quoted values in .md but parsed bare for matching"
metrics:
  duration: "~12 minutes"
  completed: "2026-04-10"
  tasks_completed: 2
  files_created: 3
---

# Phase 33 Plan 01: Core Lib Modules Summary

Three lib modules built: seed CRUD with JSONL index, requirements CRUD with atomic writes, and Kahn's algorithm wave resolution.

## Tasks Completed

| Task | Name | Commit | Files |
|------|------|--------|-------|
| 1 | Create seed-store.js | 5136af7 | lib/seed-store.js |
| 2 | Create requirements.js and wave-planner.js | 60358fa | lib/requirements.js, lib/wave-planner.js |

## What Was Built

### seed-store.js
- `nextSeedId(seedsDir)` — scans for highest SEED-NNN, returns next zero-padded ID
- `parseFrontmatter(content)` — regex YAML parser handling inline arrays, YAML lists, key:value
- `writeSeed(seedsDir, opts)` — writes SEED-NNN-slug.md then appends to index.jsonl (file first per Pitfall 1)
- `readSeed(seedPath)` — reads and parses a seed file
- `readIndex(indexPath)` / `appendToIndex(indexPath, record)` — JSONL read/append
- `surfaceMatchingSeeds(seedsDir, milestoneContext)` — case-insensitive trigger_conditions matching

### requirements.js
- `readRequirements(projectRoot)` — reads `.seraphim/requirements.json`, returns `{categories:{},reqs:{}}` on missing
- `writeRequirements(projectRoot, data)` — atomic tmp+rename to requirements.json
- `nextReqId(requirements)` — scans reqs keys, returns next REQ-NNN
- `addRequirement(projectRoot, opts)` — adds to both reqs object and categories array
- `updateRequirement(projectRoot, reqId, updates)` — merges updates into existing req

### wave-planner.js
- `resolveWaves(tasks)` — Kahn's BFS producing `[[wave0_ids], [wave1_ids], ...]`; throws on cycle with 'Circular dependency' message
- `validateDependencies(tasks)` — returns `{valid, errors}` checking depends_on refs exist
- `readWaves(projectRoot, featureId)` / `writeWaves(projectRoot, featureId, waves)` — scoped to feature objects in roadmap.json

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 1 - Bug] parseFrontmatter stripping quotes from string values**
- **Found during:** Task 1 verification
- **Issue:** YAML frontmatter writes `title: "Test Idea"` with quotes; parseFrontmatter returned `"Test Idea"` (with quotes) instead of `Test Idea`
- **Fix:** Added `.replace(/^["']|["']$/g, '')` to simple string value parsing
- **Files modified:** lib/seed-store.js
- **Commit:** 5136af7

## Known Stubs

None — all three lib modules are fully implemented with no placeholders.

## Self-Check: PASSED

- lib/seed-store.js exists: FOUND
- lib/requirements.js exists: FOUND
- lib/wave-planner.js exists: FOUND
- Commit 5136af7 exists: FOUND
- Commit 60358fa exists: FOUND
- All 7 seed-store exports verified
- All 5 requirements exports verified
- All 4 wave-planner exports verified
