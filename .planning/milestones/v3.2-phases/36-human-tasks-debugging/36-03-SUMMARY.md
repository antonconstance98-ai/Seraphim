---
phase: 36-human-tasks-debugging
plan: "03"
subsystem: commands
tags: [enrichment, auto-repair, human-tasks, pipeline, debugging]
dependency_graph:
  requires: [36-02]
  provides: [enrichment-emission, auto-repair-cascade]
  affects: [commands/run.md, commands/execute-plan.md]
tech_stack:
  added: []
  patterns: [repair-cascade, jsonl-append-log, enrichment-attrs]
key_files:
  modified:
    - path: commands/run.md (plugin)
      reason: Add enrichment field population at HUMAN_TASKS marker emission (D-02)
    - path: commands/execute-plan.md (plugin)
      reason: Add auto-repair cascade on task failure (D-05, DBG-04)
decisions:
  - "Enrichment fields omitted entirely (undefined) when not relevant — never emit empty strings (Pitfall 1)"
  - "repair-history.jsonl is caller-written (repair.js is pure logic per plan 02 decision)"
  - "PRUNE never auto-selected — requires human judgment"
  - "UAT-linked failures spawn debug sessions via --from-uat flag (DBG-02)"
metrics:
  duration_minutes: 3
  completed_date: "2026-04-10"
  tasks_completed: 1
  tasks_total: 1
  files_modified: 2
---

# Phase 36 Plan 03: Pipeline Enrichment + Auto-Repair Wiring Summary

Wired enrichment fields (skills_to_learn, thought_prompt, research_task) into HUMAN_TASKS marker emission in run.md, and integrated the selectStrategy auto-repair cascade into execute-plan.md so failed tasks attempt RETRY (2x) then DECOMPOSE (1x) before escalating to the human.

## Tasks Completed

| # | Task | Commit | Files Modified |
|---|------|--------|----------------|
| 1 | Pipeline enrichment + auto-repair wiring | b283ac8 | commands/run.md, commands/execute-plan.md |

## What Was Built

### commands/run.md — Enrichment Fields at Marker Emission (D-02)

Step 6e now instructs Claude to analyze task context and populate three optional enrichment attrs before emitting each HUMAN_TASKS marker:

- `skills_to_learn`: comma-separated skills the human should learn (e.g. "TypeScript,SQL")
- `thought_prompt`: a question about strategic implications
- `research_task`: a specific external research action

Each field is omitted entirely (not emitted as empty string) when not relevant. The emitMarker call conditionally attaches attrs only when present, matching the Pitfall 1 constraint on comma-separated skills_to_learn.

### commands/execute-plan.md — Auto-Repair Cascade on Task Failure (D-05, DBG-04)

Step 3c now implements a full repair cascade when verification fails:

1. Reads `.planning/debug/repair-history.jsonl` for prior attempts on the same task_id
2. Calls `selectStrategy(task, history)` from `lib/repair.js`
3. Executes the returned strategy:
   - **RETRY**: Re-runs task up to 2 times; appends attempt to repair-history.jsonl
   - **DECOMPOSE**: Splits task into 2-3 subtasks; attempts each sequentially
   - **ESCALATE**: Surfaces to human with full prior attempt context
4. DBG-02: If failure is from UAT context, spawns `/seraphim:debug {slug} --from-uat`
5. PRUNE documented but never auto-selected

## Deviations from Plan

None — plan executed exactly as written.

## Self-Check: PASSED

- commands/run.md: FOUND
- commands/execute-plan.md: FOUND
- Commit b283ac8: FOUND
