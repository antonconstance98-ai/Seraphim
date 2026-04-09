---
phase: 02-progress-visibility
plan: 04
subsystem: sync-pipeline
tags: [push-client, pm-sync, cost-trend, hooks, commands]
dependency_graph:
  requires: [02-03]
  provides: [pm-sync-pipeline, auto-push-pm-files, manual-sync-command]
  affects: [dashboard-ingest, phase-push-hook]
tech_stack:
  added: []
  patterns: [fire-and-forget, cross-project-aggregation, hook-extension]
key_files:
  created:
    - ~/.claude/plugins/seraphim/commands/sync.md
  modified:
    - ~/.claude/plugins/seraphim/lib/push-client.js
    - ~/.claude/plugins/seraphim/hooks/phase-push.js
decisions:
  - "pushPmData derives project root from filePath in phase-push.js using regex strip of /.seraphim/.*$"
  - "aggregateCostByDate reads full decisions.jsonl (not last 200 lines) for complete trend history"
  - "sync.md uses setTimeout(2000) to give fire-and-forget fetch time before process exits"
metrics:
  duration: 3 min
  completed_date: "2026-04-09"
  tasks_completed: 2
  tasks_total: 2
  files_changed: 3
---

# Phase 02 Plan 04: PM Sync Pipeline Summary

**One-liner:** PM data sync pipeline with pushPmData (roadmap+features+tasks+cost trend), auto-trigger on roadmap.json/task-completions.jsonl changes, and manual /seraphim:sync command.

## What Was Built

### Task 1: pushPmData in push-client.js

Added three new functions to `~/.claude/plugins/seraphim/lib/push-client.js`:

- `scanPendingTasks(projectRoot, projectName)` — reads task-completions.jsonl for completed IDs, scans all forge-log.md files for HUMAN_TASK markers (regex attribute parsing), returns pending human tasks as `{task_id, type, status, feature_id, urgency}` objects.

- `aggregateCostByDate(projectRoots)` — reads full decisions.jsonl (not last 200 lines) from every project root, groups by YYYY-MM-DD date, caps at last 365 days, returns `{date, cost_usd, project_name}` array.

- `pushPmData(projectRoot)` — fire-and-forget function that reads roadmap.json, builds milestones + features arrays, calls scanPendingTasks and aggregateCostByDate across all discovered projects, then POSTs the complete PM payload to `/api/ingest`. Silently skips if SERAPHIM_DASHBOARD_URL is not set. Existing `pushProjectData` is unchanged.

module.exports updated to `{ pushProjectData, pushPmData }`.

### Task 2: phase-push.js extension + /seraphim:sync command

Extended `hooks/phase-push.js`:
- Added `isPmOutput` condition matching `.seraphim/roadmap.json` and `.seraphim/task-completions.jsonl`
- Guard changed from `if (!isPhaseOutput)` to `if (!isPhaseOutput && !isPmOutput)`
- When `isPmOutput` is true: derives project root by stripping `/.seraphim/.*` from filePath, calls `pushPmData(pmProjectRoot)` fire-and-forget, returns early
- Existing `isPhaseOutput` → `pushProjectData` path is unchanged

Created `commands/sync.md`:
- Frontmatter: `description="Manually sync PM data to Neon dashboard"`, `allowed-tools=["Read", "Bash"]`
- Step 1: checks SERAPHIM_DASHBOARD_URL, prints friendly message if missing
- Step 2: walks up from cwd to find project root via `.seraphim/config.json`
- Step 3: runs `pushPmData` via node inline script with setTimeout(2000) to allow async fetch
- Step 4: reports result to user

## Deviations from Plan

None — plan executed exactly as written.

## Known Stubs

- `cost_usd: 0` in milestones and features arrays — cost per milestone/feature is not yet tracked in roadmap.json. The field is included in the payload shape but always zero. A future plan can wire actual cost from decisions.jsonl grouped by feature_id when that tracking is added.

## Self-Check: PASSED

- `~/.claude/plugins/seraphim/lib/push-client.js` — modified, contains pushPmData, scanPendingTasks, aggregateCostByDate
- `~/.claude/plugins/seraphim/hooks/phase-push.js` — modified, contains isPmOutput
- `~/.claude/plugins/seraphim/commands/sync.md` — created
- Commit 736cee7 (Task 1) — verified in plugin repo
- Commit 0a2d51d (Task 2) — verified in plugin repo
