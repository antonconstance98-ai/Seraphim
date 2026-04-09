---
phase: quick
plan: 260409-kam
type: execute
subsystem: seraphim-plugin-commands
tags: [milestone-lifecycle, commands, seraphim]
key-files:
  created:
    - ~/.claude/plugins/seraphim/commands/new-milestone.md
  modified:
    - ~/.claude/plugins/seraphim/commands/new-project.md
    - ~/.claude/plugins/seraphim/commands/add-feature.md
    - ~/.claude/plugins/seraphim/commands/close-milestone.md
decisions:
  - Inline milestone creation in add-feature uses same node script pattern as new-milestone for consistency
  - new-project uses milestone version ms-001 directly (no max-scan needed — always first milestone)
metrics:
  duration: ~8 min
  completed: 2026-04-09
  tasks_completed: 3
  tasks_total: 3
---

# Quick Task 260409-kam: New-project and New-milestone Commands Summary

**One-liner:** Completed milestone lifecycle with new-milestone command, project description collection in new-project, active milestone guard in add-feature, and next-milestone prompt in close-milestone.

## Tasks Completed

| # | Task | Commit |
|---|------|--------|
| 1 | Create new-milestone.md + update new-project.md | b02a433 |
| 2 | Refactor add-feature.md with active milestone guard | 48bd89b |
| 3 | Extend close-milestone.md with next milestone prompt | a8726c4 |

## What Was Built

**new-milestone.md** — Standalone `/seraphim:new-milestone` command. Resolves project root, collects version + name interactively, writes milestone to roadmap.json using readRoadmap/writeRoadmap, detects duplicate versions, prints confirmation with next-step hint.

**new-project.md** — Updated to collect `project_description` (written to config.json) and first milestone (version + name) during init. Step 4b writes `roadmap.json` with ms-001. Step 6 summary now shows milestone details. Next steps now point to `/seraphim:add-feature` instead of `/seraphim:run`.

**add-feature.md** — Step 3 Milestone section replaced with active milestone guard. Reads roadmap, filters to non-complete/non-archived milestones. If none active: offers inline creation (collect version + name, write to roadmap) or directs to `/seraphim:new-milestone`. If active exist and --milestone not provided: lists only active milestones. If --milestone provided: validates it is active, warns with status if complete/archived.

**close-milestone.md** — New Step 5 added after archival. Checks remaining active milestones. If HAS_ACTIVE: prints version list, no prompt. If NO_ACTIVE: asks to create next milestone; if yes, collects version + name, writes to roadmap, prints confirmation.

## Deviations from Plan

None — plan executed exactly as written.

## Self-Check

- [x] `~/.claude/plugins/seraphim/commands/new-milestone.md` exists
- [x] `~/.claude/plugins/seraphim/commands/new-project.md` contains "milestone version"
- [x] `~/.claude/plugins/seraphim/commands/add-feature.md` contains "active milestone" and "new-milestone"
- [x] `~/.claude/plugins/seraphim/commands/close-milestone.md` contains "next milestone" and "new-milestone"
- [x] All commits exist: b02a433, 48bd89b, a8726c4

## Self-Check: PASSED
