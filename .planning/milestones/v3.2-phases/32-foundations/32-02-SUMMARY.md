---
phase: 32-foundations
plan: "02"
subsystem: decisions-pipeline
tags: [feature_id, decisions, ingest, crucible, judge, roadmap]
dependency_graph:
  requires: [32-01]
  provides: [FOUND-03]
  affects: [dashboard/lib/types.ts, dashboard/app/api/ingest/route.ts, commands/crucible.md, commands/judge.md]
tech_stack:
  added: []
  patterns: [readRoadmap for active feature lookup, feature_id null-safe fallback]
key_files:
  created: []
  modified:
    - /home/alucardmessangeroflight/.claude/plugins/seraphim/dashboard/lib/types.ts
    - /home/alucardmessangeroflight/.claude/plugins/seraphim/dashboard/app/api/ingest/route.ts
    - /home/alucardmessangeroflight/.claude/plugins/seraphim/commands/crucible.md
    - /home/alucardmessangeroflight/.claude/plugins/seraphim/commands/judge.md
decisions:
  - feature_id resolves active roadmap feature via readRoadmap in-progress status check, not pipeline phase string
  - Graceful null fallback in try/catch when roadmap.json is absent or unreadable
metrics:
  duration: ~5 min
  completed: "2026-04-10"
  tasks_completed: 2
  tasks_total: 2
  files_modified: 4
requirements: [FOUND-03]
---

# Phase 32 Plan 02: feature_id Pipeline Wiring Summary

Wire feature_id through the decisions pipeline so every decision logged to Neon carries the actual roadmap feature ID (feat-NNN), not the pipeline phase string.

## Tasks Completed

| Task | Name | Commit | Files |
|------|------|--------|-------|
| 1 | Add feature_id to Decision type and ingest route | 9c07ba7 | dashboard/lib/types.ts, dashboard/app/api/ingest/route.ts |
| 2 | Fix feature_id source in crucible.md and judge.md | cd4f9b4 | commands/crucible.md, commands/judge.md |

## What Was Done

**Task 1** added `feature_id?: string | null` to the `Decision` interface in `types.ts` after the `loop_count` field. The decisions INSERT in `ingest/route.ts` was updated to include `feature_id` in both the column list and the VALUES clause (`${d.feature_id ?? null}`).

**Task 2** fixed both `crucible.md` and `judge.md`. Both had `feature_id: phaseId` in their `buildRecord()` calls — passing the pipeline phase string ("crucible", "judge") instead of the roadmap feature ID. The fix adds a `readRoadmap()` lookup block before each `buildRecord()` call that iterates `roadmap.milestones[].features[]` looking for `feat.status === 'in-progress'`. The resolved ID (or `null` if roadmap is unavailable) replaces `phaseId`.

## Deviations from Plan

None — plan executed exactly as written.

## Known Stubs

None. `feature_id` is fully wired end-to-end: crucible/judge resolve it from roadmap, decisions-logger accepts it (null-default added in plan 32-01), and the ingest route now includes it in the Neon INSERT.

## Self-Check

- [x] `grep "feature_id?: string | null" dashboard/lib/types.ts` — FOUND
- [x] `grep "feature_id" dashboard/app/api/ingest/route.ts` — 2 lines in decisions INSERT (column + value)
- [x] `grep -c "feature_id: phaseId" commands/crucible.md` — 0
- [x] `grep -c "feature_id: phaseId" commands/judge.md` — 0
- [x] `grep -c "readRoadmap" commands/crucible.md` — 2
- [x] `grep -c "readRoadmap" commands/judge.md` — 2
- [x] Commits 9c07ba7 and cd4f9b4 exist in plugin repo

## Self-Check: PASSED
