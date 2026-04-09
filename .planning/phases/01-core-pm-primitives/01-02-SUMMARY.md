---
phase: 01-core-pm-primitives
plan: 02
subsystem: pm-commands
tags: [slash-commands, feature-lifecycle, wip-management, roadmap]
dependency_graph:
  requires: ["01-01"]
  provides: ["add-feature-command", "start-command"]
  affects: ["roadmap.json", "pipeline-launch"]
tech_stack:
  added: []
  patterns: ["interactive-prompt-md-commands", "node-e-inline-scripts", "walk-up-project-root"]
key_files:
  created:
    - ~/.claude/plugins/seraphim/commands/add-feature.md
    - ~/.claude/plugins/seraphim/commands/start.md
  modified: []
decisions:
  - "WIP check is warn-not-block per D-06: feature starts even when limit exceeded, developer is informed"
  - "Feature reordering documented as direct JSON edit per QUEUE-04 minimal requirement"
  - "start.md delegates pipeline launch via /seraphim:run using the feature slug as phase-id"
metrics:
  duration: "8 min"
  completed: "2026-04-09"
  tasks_completed: 2
  tasks_total: 2
  files_created: 2
  files_modified: 0
---

# Phase 01 Plan 02: Feature Lifecycle Commands Summary

Interactive feature creation and pipeline-start commands for the Seraphim roadmap system.

## What Was Built

Two slash commands completing the backlog-to-pipeline lifecycle:

**`/seraphim:add-feature`** — interactive feature creation with auto-generated IDs (`feat-NNN`),
milestone lookup/creation, optional slug generation, and atomic `roadmap.json` persistence.
No sprint or time-boxing concepts. Includes reordering documentation (direct JSON edit, QUEUE-04).

**`/seraphim:start`** — feature status transition to `in-progress` with WIP limit warning (not
blocking), milestone auto-promotion from `planned` to `in-progress`, and delegation to
`/seraphim:run {slug}` to launch the six-phase pipeline.

## Tasks Completed

| Task | Name | Commit | Files |
|------|------|--------|-------|
| 1 | Create /seraphim:add-feature | 147e5ff | commands/add-feature.md |
| 2 | Create /seraphim:start | d84b1b5 | commands/start.md |

## Decisions Made

- **WIP warn-not-block**: D-06 required a warning, not a hard stop. Both `currentWip` and `maxWip`
  are printed so the developer can act if needed. Feature always starts.
- **Slug as pipeline phase-id**: `start.md` passes `feature.slug` to `/seraphim:run` as the
  phase-id — consistent with how `run.md` expects a slug-style argument (e.g., `01-add-auth`).
- **Milestone auto-creation**: `add-feature.md` creates a new milestone if the requested version
  doesn't exist, using incrementing `ms-NNN` IDs. Prevents blocking when roadmap is empty.

## Deviations from Plan

None — plan executed exactly as written.

## Known Stubs

None — both commands fully implement their specified behavior.

## Self-Check: PASSED

- `~/.claude/plugins/seraphim/commands/add-feature.md` — FOUND
- `~/.claude/plugins/seraphim/commands/start.md` — FOUND
- Commit 147e5ff — FOUND (feat(01-02): create /seraphim:add-feature slash command)
- Commit d84b1b5 — FOUND (feat(01-02): create /seraphim:start slash command)
