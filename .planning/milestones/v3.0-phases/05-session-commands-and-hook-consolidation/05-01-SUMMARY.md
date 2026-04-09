---
phase: 05-session-commands-and-hook-consolidation
plan: 01
subsystem: commands
tags: [seraphim, commands, help, status, executor-availability, profiles]

requires:
  - phase: 01-plugin-scaffold-and-infrastructure
    provides: plugin structure, config.js, phase-state.js, command file pattern
  - phase: 02-model-executors-and-pricing
    provides: executor available() API for all five external executors

provides:
  - /seraphim:help — full command reference with live profile enumeration from profiles.json
  - /seraphim:status — live state snapshot with per-executor availability probes

affects: [all phases, user-facing commands, session management]

tech-stack:
  added: []
  patterns:
    - "Command file pattern: YAML frontmatter (description, argument-hint, allowed-tools) + Claude prompt body"
    - "Live data injection: node -e inline script reads profiles.json/models.json at runtime, no hardcoded values"
    - "Project root walk-up: bash while loop searching for .seraphim/config.json"

key-files:
  created:
    - ~/.claude/plugins/seraphim/commands/help.md
    - ~/.claude/plugins/seraphim/commands/status.md
  modified: []

key-decisions:
  - "help.md enumerates profiles live from profiles.json — never hardcodes profile names"
  - "status.md probes all five executors (codex-exec, minimax-exec, gemini-exec, qwen-exec, perplexity-exec) via available() — catches auth and network failures at status-check time"
  - "status.md handles missing project root gracefully — still prints model availability when no .seraphim/config.json found"

patterns-established:
  - "All seraphim commands follow run.md pattern: YAML frontmatter + numbered step prompt body"
  - "Utility commands (help, status) use allowed-tools: [Read, Bash] — no Write or Edit needed"

requirements-completed: [SESS-01, SESS-05]

duration: 8min
completed: 2026-04-08
---

# Phase 05 Plan 01: Session Commands (help + status) Summary

**Two read-only utility commands: /seraphim:help with live profile enumeration from profiles.json, and /seraphim:status with per-executor availability probes for all five external models**

## Performance

- **Duration:** ~8 min
- **Started:** 2026-04-08T22:52:00Z
- **Completed:** 2026-04-08T23:00:00Z
- **Tasks:** 2
- **Files modified:** 2

## Accomplishments
- help.md prints all 14 commands in three sections (Pipeline, Session, Config) with live profile tables read from profiles.json at runtime
- status.md probes all five external executors via available() and displays live availability alongside config snapshot and optional phase state
- Both files handle edge cases: help.md enumerates profiles dynamically, status.md degrades gracefully when no project root is found

## Task Commits

1. **Task 1: Create help.md** - `b0ab8ac` (feat — part of combined commit)
2. **Task 2: Create status.md** - `b0ab8ac` (feat — combined with Task 1)

## Files Created/Modified
- `~/.claude/plugins/seraphim/commands/help.md` — Static command reference with live profile/model enumeration
- `~/.claude/plugins/seraphim/commands/status.md` — Live state snapshot with executor availability probes

## Decisions Made
- help.md reads profiles.json at runtime via `node -e` inline script so new profiles appear automatically without editing the command file
- status.md uses `2>/dev/null` on the executor probe to suppress stderr from missing env vars — output is captured as structured JSON only
- Both commands use `allowed-tools: ["Read", "Bash"]` — no Write/Edit needed for read-only utility commands

## Deviations from Plan
None — plan executed exactly as written.

## Issues Encountered
None.

## User Setup Required
None — no external service configuration required.

## Next Phase Readiness
- /seraphim:help and /seraphim:status are immediately usable in Claude Code
- Remaining Phase 05 plans: pause.md, resume.md, and hook consolidation
- No blockers for Phase 05 Plan 02

---
*Phase: 05-session-commands-and-hook-consolidation*
*Completed: 2026-04-08*
