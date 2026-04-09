---
phase: 01-core-pm-primitives
plan: "01"
subsystem: pm-primitives
tags: [roadmap, config, decisions-logger, slash-command]
dependency_graph:
  requires: []
  provides: [roadmap.js, /seraphim:roadmap command]
  affects: [all subsequent PM commands]
tech_stack:
  added: []
  patterns: [atomic-temp-file-rename, graceful-missing-file-fallback]
key_files:
  created:
    - ~/.claude/plugins/seraphim/lib/roadmap.js
    - ~/.claude/plugins/seraphim/commands/roadmap.md
  modified:
    - ~/.claude/plugins/seraphim/lib/config.js
    - ~/.claude/plugins/seraphim/lib/decisions-logger.js
decisions:
  - "readRoadmap returns { milestones: [] } on missing file rather than throwing (D-08)"
  - "writeRoadmap uses atomic temp-file rename to prevent corruption on concurrent writes"
  - "findFeature accepts both feat-NNN ID and slug to support human-readable references (D-01)"
  - "feature_id defaults to null in buildRecord so all existing callers remain unaffected (D-09)"
metrics:
  duration: "~8 minutes"
  completed_date: "2026-04-09"
  tasks_completed: 2
  files_created: 2
  files_modified: 2
---

# Phase 01 Plan 01: Core PM Primitives — Foundation Library Summary

**One-liner:** Foundational roadmap.js CRUD library with atomic writes and slug/ID lookup, plus config max_wip and decisions feature_id extensions.

## What Was Built

### roadmap.js (new)

Six exported functions that every subsequent PM plan depends on:

- `readRoadmap(projectRoot)` — reads `.seraphim/roadmap.json`; returns `{ milestones: [] }` if missing (D-08 compliance)
- `writeRoadmap(projectRoot, data)` — atomic write via `roadmap.json.tmp` then `renameSync` to prevent corruption on concurrent writes
- `findFeature(roadmap, ref)` — resolves both `feat-NNN` IDs and slug strings; returns `{ feature, milestone }` or `null` (D-01 compliance)
- `nextFeatureId(roadmap)` — scans all milestones for highest `feat-NNN` suffix, returns next zero-padded ID
- `countWip(roadmap)` — integer count of `in-progress` features across all milestones
- `milestoneProgress(milestone)` — returns `{ complete, total, percent }` from feature statuses

### config.js (extended)

Added `max_wip: 2` to `CONFIG_DEFAULTS` after `max_loops`. No other changes. All existing callers unaffected.

### decisions-logger.js (extended)

Added `feature_id = null` parameter to `buildRecord()` destructured args. Added `feature_id` field to returned record after `loop_count`. Existing callers that don't pass `feature_id` receive `null` automatically — zero breaking changes.

### /seraphim:roadmap command (new)

Slash command that:
- Resolves project root by walking up from cwd looking for `.seraphim/config.json`
- Reads roadmap via `readRoadmap`; prints empty-state message if no milestones
- Renders milestone-feature tree with status icons (`[●]` in-progress, `[✓]` complete, `[ ]` planned, `[!]` blocked) and progress percentages
- Filters to non-complete milestones by default; `--all` flag shows everything
- Prints summary totals line (planned / in-progress / complete)

## Commits

| Task | Commit | Description |
|------|--------|-------------|
| 1 | 9221629 | feat(01-01): create roadmap.js library and extend config/decisions-logger |
| 2 | dab3a97 | feat(01-01): create /seraphim:roadmap slash command |

## Deviations from Plan

None — plan executed exactly as written.

## Known Stubs

None. All functions are fully implemented with real logic.

## Self-Check: PASSED

- `/home/alucardmessangeroflight/.claude/plugins/seraphim/lib/roadmap.js` — exists, verified loadable
- `/home/alucardmessangeroflight/.claude/plugins/seraphim/commands/roadmap.md` — exists, verified grep checks pass
- Commits 9221629, dab3a97 — verified in plugin repo log
