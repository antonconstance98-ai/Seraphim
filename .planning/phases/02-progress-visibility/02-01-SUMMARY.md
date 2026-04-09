---
phase: 02-progress-visibility
plan: "01"
subsystem: overview-command
tags: [pm, overview, cross-project, scanner]
dependency_graph:
  requires: []
  provides: [seraphim-overview-command, readPmSummary-export]
  affects: [multi-project-scanner, commands-directory]
tech_stack:
  added: []
  patterns: [node-inline-script-command, lazy-require-helpers]
key_files:
  created:
    - ~/.claude/plugins/seraphim/commands/overview.md
  modified:
    - ~/.claude/plugins/seraphim/lib/multi-project-scanner.js
decisions:
  - readPmSummary lazy-loads roadmap/config/markers to avoid circular deps
  - pendingGates counted from TASK markers with status=pending in forge-log.md
  - idle filter: wip===0 AND pendingTasks===0 AND pendingGates===0 (per OVER-03)
metrics:
  duration: "~5 min"
  completed_date: "2026-04-09T18:02:52Z"
  tasks_completed: 2
  files_changed: 2
---

# Phase 02 Plan 01: Cross-Project Overview Command Summary

**One-liner:** `/seraphim:overview` command with readPmSummary showing active projects, WIP limits, blocked features, and pending gates — idle-filtered by default with `--all` flag.

## What Was Built

### Task 1 — readPmSummary added to multi-project-scanner.js (commit 4091631)

Added `readPmSummary(projectRoot)` function that returns a PM summary object:
- `projectName` — from `path.basename(projectRoot)`
- `activeMilestone` — first non-complete milestone with version, name, percent
- `wip` / `wipLimit` — from roadmap.js countWip and config.js max_wip (default 2)
- `pendingTasks` — count of unresolved human TASK markers in forge-log.md files
- `blockedFeatures` — feature IDs where depends_on contains an incomplete dep
- `pendingGates` — TASK markers with status 'pending'

Helper libs (roadmap, config, markers) are lazy-loaded via getter functions to avoid circular dependency issues.

### Task 2 — /seraphim:overview command (commit ceae777)

Created `commands/overview.md` with:
- Frontmatter: description, argument-hint `[--all]`, allowed-tools
- `--all` flag parsing controlling idle project filter
- Node script calling `discoverSeraphimProjects` then `readPmSummary` per project
- Idle filter: `wip===0 AND pendingTasks===0 AND pendingGates===0`
- NEEDS ATTENTION section at top (D-01) with blocked feature, WIP exceeded, and pending gate signals
- PROJECT / MILESTONE / WIP / TASKS column table
- Footer shown when idle projects exist and `--all` is not set

## Deviations from Plan

None — plan executed exactly as written.

## Known Stubs

None — all fields are sourced from real data on disk.

## Self-Check: PASSED

- `~/.claude/plugins/seraphim/commands/overview.md` — exists (created)
- `~/.claude/plugins/seraphim/lib/multi-project-scanner.js` — modified with readPmSummary
- Commit 4091631 — verified in plugin repo
- Commit ceae777 — verified in plugin repo
- `node -e "... Object.keys(s)"` returns `['discoverSeraphimProjects', 'readProjectMeta', 'readPmSummary']` ✓
- `grep 'NEEDS ATTENTION' overview.md` — returns match ✓
- `grep 'readPmSummary' overview.md` — returns match ✓
