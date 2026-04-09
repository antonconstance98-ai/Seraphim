---
phase: 01-core-pm-primitives
plan: "04"
subsystem: session-management
tags: [pause-resume, milestone-archival, pm-context, cost-tracking]
dependency_graph:
  requires: ["01-01", "01-02"]
  provides: ["PM context persistence via state.pm", "milestone archival via close-milestone.md"]
  affects: ["session continuity", "roadmap.json active milestones"]
tech_stack:
  added: []
  patterns: ["atomic temp+rename write", "roadmap.js for in-progress feature scan", "decisions.jsonl cost aggregation"]
key_files:
  created:
    - ~/.claude/plugins/seraphim/commands/close-milestone.md
  modified:
    - ~/.claude/plugins/seraphim/commands/pause.md
    - ~/.claude/plugins/seraphim/commands/resume.md
decisions:
  - "PM context block (state.pm) is null when no active feature — backward compat for pre-PM sessions"
  - "close-milestone warns on incomplete milestone; --force overrides — never silently archive partial work"
  - "cost aggregation reads decisions.jsonl at archive time — no running total required"
metrics:
  duration: "~6 minutes"
  completed_date: "2026-04-09"
  tasks_completed: 2
  tasks_total: 2
  files_created: 1
  files_modified: 2
---

# Phase 01 Plan 04: PM Context Persistence and Milestone Archival Summary

**One-liner:** PM context (feature_id, milestone_version, progress) now survives pause/resume via state.json, and close-milestone.md archives completed milestones with decisions.jsonl cost breakdown.

## What Was Built

### Task 1: pause.md and resume.md extended with PM context block

`pause.md` Step 3 now requires `roadmap.js` to scan for the first `in-progress` feature across all milestones. If found, it writes `state.pm = { feature_id, milestone_version, progress: { phase, status: 'paused' } }` to state.json before calling `ps.writeState()`. If no in-progress feature exists, `state.pm = null` — pre-PM sessions are unaffected.

`resume.md` Step 3 now reads and prints the PM context block when present. Step 4 deletes `state.pm` after restoring it (clean state after resume). Step 5 banner shows a `Feature:` line only when PM context was present.

### Task 2: /seraphim:close-milestone command

New file `commands/close-milestone.md` implements milestone archival:

- Finds milestone by version in roadmap.json
- Warns if completion < 100% (exit 1); `--force` overrides
- Reads `decisions.jsonl`, filters records by `feature_id` membership in the milestone
- Sums `cost_usd` per feature and totals
- Writes archive JSON to `.seraphim/milestones/{version}.json` via atomic temp+rename
- Archive schema: `{ version, name, archived_at, completion_percent, features, cost_usd, cost_breakdown }`
- Removes the milestone from `rm.milestones` and calls `writeRoadmap()` to persist
- Missing roadmap prints "No roadmap found." and exits 0 (never errors)

## Deviations from Plan

### Auto-fixed Issues

None — plan executed exactly as written.

## Known Stubs

None — all data flows are wired. Cost aggregation reads live from decisions.jsonl at archive time.

## Self-Check

- pause.md contains `state.pm`: verified via grep
- resume.md contains `state.pm`: verified via grep
- close-milestone.md exists with `milestones/`, `cost_usd`, `decisions.jsonl`, `archived_at`: verified via grep
- Task 1 commit `c8218bd` exists in plugin repo
- Task 2 commit `bce81fc` exists in plugin repo
