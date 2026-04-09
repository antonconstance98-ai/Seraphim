---
phase: 04-quality-gates-and-decision-logging
plan: "04"
subsystem: crucible-logging-and-session-validation
tags: [decisions-logging, quality-gates, session-start, crucible, fix-instructions]
dependency_graph:
  requires: [04-01]
  provides: [crucible-decisions-logging, session-start-validator, fix-instructions-section]
  affects: [forge-fix-mode, decisions-integrity]
tech_stack:
  added: []
  patterns: [fail-open-validation, decisions-jsonl-append, loop-cap-escalation]
key_files:
  modified:
    - ~/.claude/plugins/seraphim/commands/crucible.md
  created:
    - ~/.claude/plugins/seraphim/hooks/session-start.js
key_decisions:
  - "session-start.js created from scratch — file was absent from plugin hooks directory (Rule 3 auto-fix)"
  - "pluginRoot resolved from CLAUDE_PLUGIN_ROOT env with fallback to __dirname/../ — consistent with token-logger.js pattern"
  - "projectRoot walk-up searches 10 levels for .seraphim/config.json — silently skips validator if no Seraphim project found"
metrics:
  duration_min: 8
  completed_date: "2026-04-08"
  tasks_completed: 2
  files_changed: 2
---

# Phase 04 Plan 04: Crucible Decisions Logging and Session-Start Validator Summary

Crucible now appends a decisions.jsonl record with crucible_pass_rate signal after every run, outputs a structured Fix Instructions section when verdict=fail, surfaces last crucible.md findings in loop-cap escalation, and sessions start with a fail-open integrity check against decisions.jsonl.

## Tasks Completed

| Task | Name | Commit | Files |
|------|------|--------|-------|
| 1 | Add Fix Instructions section and decisions.jsonl logging to crucible.md | de2fae6 | commands/crucible.md |
| 2 | Wire decisions-validator.js into session-start hook | 3a90266 | hooks/session-start.js |

## What Was Built

**Task 1 — crucible.md (three changes):**

1. `startMs = Date.now()` recorded at the top of Step 3 so `latency_ms` captures full crucible execution time.
2. Step 5 loop-cap escalation now reads the last crucible.md, extracts Issues Found and Vulnerabilities Found sections, surfaces them in the error message, and lists three concrete manual resolution options (address directly / update blueprint / reset loop counter).
3. Step 9 output template now includes a `## Fix Instructions` section with `SERAPHIM:FIX_INSTRUCTIONS` marker, generated only when `verdict === 'fail'`. Lists targeted fix tasks by task ID for Forge to consume in fix mode.
4. Step 10b added after the banner: calls `appendDecision` via decisions-logger.js with `crucible_pass_rate: 1.0` (pass) or `0.0` (fail), summed costs from both verify and adversarial passes, and full quality_signals block.

**Task 2 — session-start.js (created):**

New hook file that runs on `SessionStart`. Resolves `projectRoot` by walking up from event `cwd` searching for `.seraphim/config.json`. If a Seraphim project is found, calls `validateDecisions(projectRoot)`. Violations are printed to stderr with `[seraphim] decisions.jsonl integrity check FAILED:` prefix. If no Seraphim project found, skips silently. Entire validator call is wrapped in try/catch — hook never throws, never calls `process.exit()`.

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 3 - Blocking] session-start.js did not exist**
- **Found during:** Task 2
- **Issue:** Plan specified editing session-start.js but the file was absent from the hooks directory. Only token-logger.js existed in hooks/.
- **Fix:** Created session-start.js from scratch following the same stdin/stdout event pattern as token-logger.js, with the validator call added per the plan's specification.
- **Files modified:** hooks/session-start.js (created)
- **Commit:** 3a90266

## Requirements Addressed

- QUAL-04: Fix Instructions section with task IDs enables forge.md fix mode targeting
- QUAL-05: Loop-cap escalation surfaces previous crucible.md findings for manual resolution
- COST-03: crucible.md appends decisions.jsonl record after every run
- COST-04: crucible_pass_rate quality signal written (1.0 pass / 0.0 fail)
- COST-05: decisions-validator wired into session-start hook with fail-open behavior

## Self-Check: PASSED

- [x] `~/.claude/plugins/seraphim/commands/crucible.md` — modified, verified with grep
- [x] `~/.claude/plugins/seraphim/hooks/session-start.js` — created, verified with grep
- [x] Plugin repo commit de2fae6 — crucible.md changes
- [x] Plugin repo commit 3a90266 — session-start.js creation
- [x] `appendDecision` appears 2 times in crucible.md
- [x] `Fix Instructions` appears 1 time in crucible.md
- [x] `validateDecisions` appears 2 times in session-start.js
- [x] `crucible_pass_rate` appears 1 time in crucible.md
