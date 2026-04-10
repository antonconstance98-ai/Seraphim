---
phase: 34
plan: "03"
subsystem: navigation
tags: [commands, routing, navigation, progress, state-machine]
dependency_graph:
  requires: [33-05]
  provides: [NAV-01, NAV-02, NAV-03]
  affects: [user-workflow, command-discoverability]
tech_stack:
  added: []
  patterns: [artifact-presence-check, keyword-router, ascii-progress-bars]
key_files:
  created:
    - ~/.claude/plugins/seraphim/commands/next.md
    - ~/.claude/plugins/seraphim/commands/do.md
    - ~/.claude/plugins/seraphim/commands/progress.md
  modified: []
decisions:
  - "next.md uses dynamic directory glob (not hardcoded paths) per pitfall-4 in RESEARCH.md"
  - "do.md never auto-executes — always presents suggestion and waits for user confirmation"
  - "progress.md reads live phase directory counts, not ROADMAP.md plan counts, for accuracy"
metrics:
  duration: "~10 min"
  completed: "2026-04-10"
  tasks: 2
  files: 3
---

# Phase 34 Plan 03: Navigation Routing Summary

Three navigation commands that eliminate cognitive load: state-machine routing to next command, freeform text-to-command routing, and phase completion overview with progress bars.

## Tasks Completed

| # | Name | Commit | Files |
|---|------|--------|-------|
| 1 | Create next.md state-machine navigation command | 895d61a | commands/next.md |
| 2 | Create do.md keyword router and progress.md overview | 895d61a | commands/do.md, commands/progress.md |

## What Was Built

**next.md** — Reads STATE.md to extract current phase number, dynamically discovers the phase directory (no hardcoded paths), then checks artifact presence in strict order: CONTEXT.md → PLAN.md → SUMMARY.md → VERIFICATION.md. Stops at first missing artifact and suggests the correct command. If phase is complete, advances to next phase and repeats. Never auto-executes.

**do.md** — Parses `$ARGUMENTS` as freeform text, applies a 12-group keyword map covering all major Seraphim commands (plan/execute/debug/research/seed/progress/pause/resume/discuss/requirements/next/verify). Single match → confirm prompt. Multiple matches → numbered options. No match → points to `/seraphim:help`. Never auto-executes.

**progress.md** — Scans `.planning/phases/` directory live (not ROADMAP.md counts) for each phase: counts PLAN.md vs SUMMARY.md files, checks CONTEXT.md and VERIFICATION.md presence. Renders ASCII progress bar table ([====      ] 40%). Shows overall milestone completion. Suggests next action using same artifact logic as next.md.

## Deviations from Plan

None — plan executed exactly as written.

## Known Stubs

None — all three commands are fully wired. They read live filesystem state, not mock data.

## Self-Check: PASSED

Files exist:
- /home/alucardmessangeroflight/.claude/plugins/seraphim/commands/next.md ✓
- /home/alucardmessangeroflight/.claude/plugins/seraphim/commands/do.md ✓
- /home/alucardmessangeroflight/.claude/plugins/seraphim/commands/progress.md ✓

Commit 895d61a exists in plugin repo ✓
