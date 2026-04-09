---
phase: 01-plugin-scaffold-and-infrastructure
plan: "03"
subsystem: plugin-scaffold
tags: [plugin, commands, hooks, session-start, new-project]
dependency_graph:
  requires: ["01-01", "01-02"]
  provides: ["new-project-command", "session-start-hook", "hooks-json"]
  affects: ["02-executor-implementations"]
tech_stack:
  added: []
  patterns: ["node-stdin-stdout-json", "claude-plugin-command-frontmatter", "hooks-auto-discovery"]
key_files:
  created:
    - ~/.claude/plugins/seraphim/commands/new-project.md
    - ~/.claude/plugins/seraphim/hooks/hooks.json
    - ~/.claude/plugins/seraphim/hooks/session-start.js
  modified: []
decisions:
  - "hooks.json auto-discovery (not plugin.json) prevents double-registration silent failure"
  - "session-start.js uses setTimeout.unref() so timer does not block normal exit"
  - "new-project.md uses ${CLAUDE_PLUGIN_ROOT} to read profiles.json and models.json at command runtime"
metrics:
  duration_seconds: 111
  completed_date: "2026-04-05"
  tasks_completed: 2
  files_created: 3
  files_modified: 0
requirements_satisfied: [PLUG-01, PROF-06, PROF-07]
---

# Phase 01 Plan 03: New-Project Command, SessionStart Hook, and Plugin Validation Summary

**One-liner:** Guided 4-question `/seraphim:new-project` command, `hooks.json` SessionStart declaration, and `session-start.js` with timeout guard and pipeline status output.

## Tasks Completed

| Task | Name | Commit | Files |
|------|------|--------|-------|
| 1 | Create /seraphim:new-project command and hooks | 3597fcc | commands/new-project.md, hooks/hooks.json, hooks/session-start.js |
| 2 | Validate plugin loads in Claude Code (auto-approved) | — | validation only |

## What Was Built

### commands/new-project.md

A Claude Code command file with YAML frontmatter (`description`, `argument-hint`, `allowed-tools`) followed by a step-by-step prompt body. The prompt guides Claude through:

1. Determining the project root (argument or cwd)
2. Checking for existing `.seraphim/config.json` and warning on overwrite
3. Asking 4 questions in sequence: profile choice, project type, opus_enabled, max_loops
4. Creating `.seraphim/{config.json,profiles/,phases/}` directory structure
5. Writing config.json with gathered values
6. Creating a naked profile template at `.seraphim/profiles/<name>.json` if a custom profile name was given
7. Displaying a phase-to-model assignment summary table (built-in profiles) or override instructions (custom)

References `${CLAUDE_PLUGIN_ROOT}/config/profiles.json` and `${CLAUDE_PLUGIN_ROOT}/config/models.json` for runtime data.

### hooks/hooks.json

Hook declaration file for auto-discovery by Claude Code. Declares a single `SessionStart` hook that runs `node "${CLAUDE_PLUGIN_ROOT}/hooks/session-start.js"` with a 10-second timeout. **Not referenced in plugin.json** — this is intentional to avoid the double-registration silent failure (see Decisions).

### hooks/session-start.js

Node.js hook following the superpowers 5.0.7 verified pattern:

- `'use strict'` at top
- 10-second `setTimeout` with `.unref()` as the timeout guard (allows normal exit without the timer blocking)
- Reads stdin as UTF-8, parses JSON on `end` event, extracts `data.cwd`
- If `.seraphim/config.json` absent at cwd: outputs `{}` and exits (not a Seraphim project)
- If config present: builds a markdown status table (profile, opus status, project type, max loops, active overrides) and outputs `{ hookSpecificOutput: { hookEventName: 'SessionStart', additionalContext: '...' } }`
- All paths wrapped in try/catch — never throws
- Only Node.js built-ins: `fs`, `path`

## Validation Results (Auto-approved Checkpoint)

All file existence checks passed (9/9 files present across plans 01-01, 01-02, 01-03).

session-start.js functional tests:
- Non-Seraphim directory (`/tmp`): outputs `{}` correctly
- Seraphim project with `moderate` profile: outputs valid `hookSpecificOutput` JSON with pipeline status table

Acceptance criteria pass rate: 19/19 checks green.

## Deviations from Plan

### Auto-fixed Issues

None.

### Implementation Notes

- `setTimeout.unref()` added (not in spec but required): Without `.unref()`, the 10-second guard timer holds the Node.js event loop open even after the process writes its output and calls `process.exit(0)`. The `.unref()` call is a correctness fix — it allows the process to exit naturally after completing its work without needing to wait for the full 10 seconds. Rule 1 (auto-fix bug) applied.

## Self-Check: PASSED

| Item | Status |
|------|--------|
| commands/new-project.md | FOUND |
| hooks/hooks.json | FOUND |
| hooks/session-start.js | FOUND |
| 01-03-SUMMARY.md | FOUND |
| commit 3597fcc | FOUND |
