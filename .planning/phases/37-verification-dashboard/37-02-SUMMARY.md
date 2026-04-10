---
phase: 37-verification-dashboard
plan: 02
subsystem: commands
tags: [audit, uat, stats, terminal, verification]
one_liner: "Three audit/stats commands: cross-phase milestone audit with stale-verification detection, UAT scanner grouping pending/failed items by phase, and terminal project statistics"
dependency_graph:
  requires: []
  provides: [audit-milestone-command, audit-uat-command, stats-command]
  affects: [seraphim-plugin-commands]
tech_stack:
  added: []
  patterns: [atomic-tmp-rename, project-root-walk, node-inline-script]
key_files:
  created:
    - ~/.claude/plugins/seraphim/commands/audit-milestone.md
    - ~/.claude/plugins/seraphim/commands/audit-uat.md
    - ~/.claude/plugins/seraphim/commands/stats.md
  modified: []
decisions:
  - "audit-milestone.md uses atomic tmp+rename for report write — consistent with all other seraphim commands"
  - "audit-uat.md is read-only (no Write tool) — audit commands should not mutate state"
  - "stats.md is read-only — metrics are derived live from filesystem and git, not cached"
  - "Plugin repo commits used instead of project repo — command files live in ~/.claude/plugins/seraphim which is a separate git repo"
metrics:
  duration: "115 seconds"
  completed_date: "2026-04-10"
  tasks_completed: 2
  files_created: 3
  files_modified: 0
requirements: [VFY-05, VFY-06, VIZ-04]
---

# Phase 37 Plan 02: Audit and Stats Commands Summary

Three audit/stats commands: cross-phase milestone audit with stale-verification detection, UAT scanner grouping pending/failed items by phase, and terminal project statistics.

## Tasks Completed

| Task | Name | Commit | Files |
|------|------|--------|-------|
| 1 | Create audit-milestone.md and audit-uat.md | b4d91af | commands/audit-milestone.md, commands/audit-uat.md |
| 2 | Create stats.md command | ab4d7d1 | commands/stats.md |

## What Was Built

**audit-milestone.md** (`/seraphim:audit-milestone`):
- Walks all phase directories and collects VERIFICATION.md files
- Checks each verification's `verified:` timestamp against `git log -1 --format=%aI` for that phase directory — flags stale verifications
- Counts `[x]` vs `[ ]` markers in REQUIREMENTS.md for coverage percentage
- Spawns an integration-checker subagent to review cross-phase data contracts, requirement coverage, and key-link integrity
- Writes audit report atomically to `.planning/audit-milestone-{YYYY-MM-DD}.md` using tmp+rename

**audit-uat.md** (`/seraphim:audit-uat`):
- Globs all `.planning/phases/*/UAT.md` files
- Parses YAML frontmatter (status, total, tested, passed, failed) and body item blocks
- Filters for `status: pending` and `status: failed` items, groups by phase
- Displays per-phase breakdown with debug session hints when failed item notes reference a debug slug
- Read-only — no Write tool

**stats.md** (`/seraphim:stats`):
- Counts total/completed phases via directory scan (context+plans+summaries+verification presence check)
- Counts total/completed plans by PLAN.md vs SUMMARY.md file counts
- Reads REQUIREMENTS.md for `[x]`/`[ ]` coverage percentage
- Runs git metrics: total commits, first commit date, files changed since start, days elapsed, avg days per completed phase
- Formatted terminal output with all metrics; graceful fallback if no git history

## Deviations from Plan

**1. [Rule 3 - Blocking Issue] Commands committed to plugin repo, not project repo**
- **Found during:** Task 1 commit
- **Issue:** Command files live in `~/.claude/plugins/seraphim/commands/` which is outside the project git repo at `~/projects/seraphim`
- **Fix:** Committed to the plugin repo at `~/.claude/plugins/seraphim` (which is itself a git repo)
- **Files affected:** All three command files
- **Commits:** b4d91af, ab4d7d1

## Known Stubs

None — all three commands are fully specified and wire directly to live filesystem/git data sources.

## Self-Check: PASSED

- `~/.claude/plugins/seraphim/commands/audit-milestone.md` — FOUND
- `~/.claude/plugins/seraphim/commands/audit-uat.md` — FOUND
- `~/.claude/plugins/seraphim/commands/stats.md` — FOUND
- Commit b4d91af — FOUND
- Commit ab4d7d1 — FOUND
