---
phase: 05-session-commands-and-hook-consolidation
plan: 03
subsystem: hooks
tags: [retirement, settings, hooks, session-start, archive]

requires:
  - phase: 04-quality-gates-and-decision-logging
    provides: decisions-validator.js, session-start.js hook
provides:
  - retire-hooks.js atomic retirement script
  - retire-hooks.md slash command
  - Clean settings.json with only plugin hooks
  - Archive of all legacy hooks
affects: [all-sessions]

tech-stack:
  added: []
  patterns: [atomic-json-write-via-rename]

key-files:
  created:
    - ~/.claude/plugins/seraphim/tools/retire-hooks.js
    - ~/.claude/plugins/seraphim/commands/retire-hooks.md
  modified:
    - ~/.claude/settings.json

key-decisions:
  - "Atomic write via writeFileSync(tmp) + renameSync prevents corruption"
  - "codex-superpowers-plan-reviewer.js explicitly preserved"
  - "codex-multi-round-reviewer.js logged as already-gone (not error)"
  - "session-start.js added to SessionStart in same atomic write as removals"
  - "token-logger.js duplication guard prevented double-add"

patterns-established:
  - "Hook retirement: archive backups + atomic settings.json rewrite"

requirements-completed: [HOOK-01, HOOK-02, HOOK-03]

duration: 5min
completed: 2026-04-08
---

# Plan 05-03: Hook Retirement

**Atomic retirement of 6 legacy hooks from settings.json, registration of plugin hooks, and archival of all backup files — verified by human checkpoint.**

## What Was Built

- **retire-hooks.js**: Node.js script that reads settings.json, removes legacy hook entries by path match, adds plugin hooks (session-start.js, token-logger.js) with duplication guard, writes atomically via tmp+rename, and moves backup files to archive/
- **retire-hooks.md**: Slash command that runs the script with dry-run preview and confirmation

## Checkpoint Result

Human-verified: all 6 legacy hooks removed, 2 plugin hooks registered, superpowers reviewer preserved, 8 archives exist, JSON valid.

## Self-Check: PASSED
