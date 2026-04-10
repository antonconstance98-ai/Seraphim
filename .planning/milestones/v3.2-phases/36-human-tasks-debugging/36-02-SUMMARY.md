---
phase: 36-human-tasks-debugging
plan: 02
subsystem: debug-commands
tags: [debug, forensics, repair, strategy-cascade]
dependency_graph:
  requires: [33-core-lib-patterns]
  provides: [debug-persistent-state, forensics-read-only, repair-cascade]
  affects: [human-task-workflow, failed-task-recovery]
tech_stack:
  added: []
  patterns: [tmp+rename atomic write, pure logic module, restricted subagent tools]
key_files:
  created:
    - ~/.claude/plugins/seraphim/lib/repair.js
    - ~/.claude/plugins/seraphim/commands/debug.md
    - ~/.claude/plugins/seraphim/commands/forensics.md
  modified: []
decisions:
  - "repair.js is pure logic with no I/O — caller handles all state writes"
  - "PRUNE is manual-only and never auto-selected by selectStrategy"
  - "forensics.md restricts subagent to Read + read-only Bash; parent writes the report file"
metrics:
  duration: ~8min
  completed_date: "2026-04-10"
  tasks: 2
  files: 3
requirements: [DBG-01, DBG-03, DBG-04]
---

# Phase 36 Plan 02: Debug Commands and Repair Cascade Summary

**One-liner:** Persistent debug sessions with atomic state files, read-only forensics subagent, and RETRY/DECOMPOSE/ESCALATE repair cascade capped at 2 retries and 1 decompose.

## Tasks Completed

| Task | Name | Commit | Files |
|------|------|--------|-------|
| 1 | lib/repair.js strategy cascade | 70b33c0 | lib/repair.js |
| 2 | debug.md and forensics.md commands | e052115 | commands/debug.md, commands/forensics.md |

## What Was Built

### lib/repair.js (D-05)
Pure logic module exporting:
- `STRATEGIES = ['RETRY', 'DECOMPOSE', 'PRUNE', 'ESCALATE']`
- `selectStrategy(task, repairHistory)` — returns RETRY if fewer than 2 prior retries, DECOMPOSE if fewer than 1 prior decompose, otherwise ESCALATE. PRUNE is never auto-selected.
- `formatRepairReport(task, strategy, reason)` — structured markdown block for debug state files.

### commands/debug.md (D-04, D-07)
Persistent debug command that:
- Accepts a `<slug>` argument and optional `--from-uat <gap-id>` flag
- Creates `.planning/debug/{slug}.md` on first run with YAML frontmatter (status, created, updated)
- Reads and displays existing state when resuming a prior session
- Appends `## Session N` blocks with findings, hypothesis, next action
- Uses tmp+rename atomic writes (renameSync) to prevent corruption on reset
- Integrates with repair.js to suggest next repair strategy for failed tasks

### commands/forensics.md (D-06)
Read-only post-mortem command that:
- Explicitly prohibits Write, Edit, and all mutating Bash commands
- Spawns a subagent restricted to: Read, Bash (git log/diff/show, grep, cat, ls, find)
- Subagent analyzes: git timeline, error patterns, state file consistency, source file relevance
- Parent agent writes the forensics report (only allowed write) to `.planning/debug/forensics/{slug}-{timestamp}.md`
- Presents root cause hypothesis, evidence, affected files, recommended fix (does not implement)

## Verification

- `node -e selectStrategy` unit test: PASS (all 3 thresholds)
- `grep -q "debug" commands/debug.md`: PASS
- `grep -q ".planning/debug/" commands/debug.md`: PASS
- `grep -q "forensics" commands/forensics.md`: PASS
- `grep "slug" commands/debug.md`: PASS (3+ lines)
- `grep "renameSync" commands/debug.md`: PASS
- `grep "MUST NOT" commands/forensics.md`: PASS
- `grep "allowed-tools" commands/forensics.md`: PASS

## Deviations from Plan

None — plan executed exactly as written.

## Known Stubs

None — all logic is complete. The debug and forensics commands reference `.planning/debug/` which is created at runtime.

## Self-Check: PASSED

Files verified:
- FOUND: ~/.claude/plugins/seraphim/lib/repair.js
- FOUND: ~/.claude/plugins/seraphim/commands/debug.md
- FOUND: ~/.claude/plugins/seraphim/commands/forensics.md

Commits verified:
- 70b33c0 feat(36-02): add repair.js strategy cascade
- e052115 feat(36-02): add debug.md and forensics.md commands
