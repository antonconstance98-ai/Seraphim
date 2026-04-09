---
phase: 01-core-pm-primitives
plan: 05
subsystem: pipeline-orchestration
tags: [pipeline, markers, auto-complete, inbox, pm]
dependency_graph:
  requires: ["01-01", "01-02", "01-03"]
  provides: ["TASK-04-write-side", "D-03-auto-complete"]
  affects: ["run.md", "roadmap.json", "forge-log.md"]
tech_stack:
  added: []
  patterns: ["node -e inline script", "SERAPHIM HTML comment markers", "non-fatal pm writes"]
key_files:
  created: []
  modified:
    - ~/.claude/plugins/seraphim/commands/run.md
decisions:
  - "Post-crucible HUMAN_TASKS marker doubles as D-03 inbox notification — no separate notification step needed"
  - "Marker emission checks roadmap.json existence first; missing roadmap causes silent exit per D-08"
  - "Auto-complete failure is non-fatal — pipeline success is already recorded before Step 7b runs"
metrics:
  duration: ~8min
  completed: "2026-04-09"
  tasks_completed: 1
  tasks_total: 1
  files_modified: 1
---

# Phase 01 Plan 05: Pipeline Gate Markers and Auto-Complete Summary

**One-liner:** SERAPHIM:HUMAN_TASKS markers emitted at three pipeline gate points and Crucible SHIP verdict auto-completes feature in roadmap.json with inbox notification.

## What Was Built

Extended `run.md` (the full pipeline orchestrator) with two new steps that close the write side of TASK-04 and implement D-03 auto-complete:

### Step 6e — HUMAN_TASKS marker emission at gate points

Inserted after Step 6d (auto-advance) for three trigger phases:

| Phase completed | Gate name     | Task type | Description                                                   |
|-----------------|---------------|-----------|---------------------------------------------------------------|
| discover        | pre-envision  | review    | "Review discovery findings for {phase-id}"                    |
| judge           | pre-architect | decision  | "Review judgment for {phase-id} before architecture"          |
| crucible        | post-crucible | review    | "Feature {slug} completed -- review Crucible results for {id}" |

Each marker is appended to `.seraphim/phases/{phase-id}/forge-log.md` using `markers.emitMarker('HUMAN_TASKS', {...})`. The forge-log.md directory and file are created if they do not exist yet (early phases run before forge creates it). Emission is skipped silently when roadmap.json is absent (D-08).

### Step 7b — Auto-complete feature on SHIP verdict

Inserted between Step 7 (banner) and Step 8 (mark complete). Reads the Crucible verdict from `crucible.md`, finds the feature by phase-id slug via `findFeature()`, sets `feature.status = 'complete'`, and persists via `writeRoadmap()`. All failures are non-fatal per D-08.

The post-crucible HUMAN_TASKS marker (Step 6e) serves as the D-03 inbox notification with the feature slug and phase-id embedded in the description — no separate notification step is required.

## Deviations from Plan

None — plan executed exactly as written.

## Self-Check

- [x] `~/.claude/plugins/seraphim/commands/run.md` modified — FOUND
- [x] Commit b1862d9 — FOUND (plugin repo)
- [x] Verification grep passed: HUMAN_TASKS, writeRoadmap, pre-envision, pre-architect, post-crucible, auto-complete all present

## Self-Check: PASSED
