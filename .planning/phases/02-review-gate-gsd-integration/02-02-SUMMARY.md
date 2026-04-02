---
phase: "02"
plan: "02"
subsystem: gsd-integration
tags: [wave-validation, plan-review, post-tool-use, subagent-stop, background-worker, d07, d08]
dependency_graph:
  requires:
    - "02-01 (codex-exec.js runCodexExec API, codex-review-gate.js Stop hook pattern)"
  provides:
    - "GSD-01: wave-boundary Codex validation dispatched automatically on wave completion"
    - "GSD-02: background non-blocking validation via detached worker process"
    - "GSD-03: plan-phase Codex review writes REVIEWS.md matching GSD detection pattern"
    - "D-07: plan-phase finalization enforced-blocked via SubagentStop hook"
  affects:
    - "~/.claude/settings.json (PostToolUse and SubagentStop hooks added)"
    - "~/.claude/hooks/codex-wave-validator.js (PostToolUse hook — wave detection)"
    - "~/.claude/hooks/codex-wave-validator-worker.js (background Codex invocation)"
    - "~/.claude/hooks/codex-plan-reviewer.js (SubagentStop hook + standalone)"
tech_stack:
  added: []
  patterns:
    - "detached child process (spawn + unref) for non-blocking background validation"
    - "temp file for prompt passing (avoids shell-escaping issues)"
    - "sentinel JSON file for cross-session result persistence"
    - "dual-mode script (hook stdin vs CLI --phase arg)"
key_files:
  created:
    - "~/.claude/hooks/codex-wave-validator.js (383 lines)"
    - "~/.claude/hooks/codex-wave-validator-worker.js (208 lines)"
    - "~/.claude/hooks/codex-plan-reviewer.js (414 lines)"
  modified:
    - "~/.claude/settings.json (PostToolUse 3rd hook + new SubagentStop key)"
decisions:
  - "temp file for prompt: wave validator writes prompt to .planning/wave-N-prompt.tmp before spawning worker, avoids shell escaping entire multi-line prompt as argv"
  - "sentinel-before-spawn: sentinel file written before child.unref() so partial results are visible even if worker crashes"
  - "fail-open in hook mode: if Codex fails in SubagentStop, plan reviewer exits 0 and injects advisory context rather than blocking — prevents permanent planner paralysis when Codex is unavailable"
  - "120s for wave validation planner review is recent PLAN.md heuristic: files modified within 120 seconds in .planning/phases/*/ identify planner subagents without needing transcript inspection"
metrics:
  duration: "6 minutes"
  completed_date: "2026-04-02"
  tasks_completed: 2
  files_created: 3
  files_modified: 1
---

# Phase 02 Plan 02: GSD Wave-Boundary Validation and Plan-Phase Review Summary

GSD wave-boundary hook and plan-phase review gate: codex-wave-validator.js (PostToolUse) dispatches background Codex validation when a wave completes, codex-plan-reviewer.js (SubagentStop) enforces D-07 by blocking plan finalization on HIGH severity issues, and both persist results across sessions.

## What Was Built

### Task 1: codex-wave-validator.js + codex-wave-validator-worker.js

**codex-wave-validator.js** (383 lines) is a PostToolUse hook that:
- Checks every PostToolUse call for pending wave validation results (Step 1b) and surfaces them as `additionalContext`
- Detects SUMMARY.md writes in Write/Edit/Bash tool calls
- Queries `gsd-tools phase-plan-index` to check if the completed plan's wave is now fully done
- Writes a sentinel file (`.planning/wave-N-validation.json`) before spawning the worker
- Spawns `codex-wave-validator-worker.js` as a detached child process (`detached: true` + `child.unref()`) per D-08 (non-blocking)
- Surfaces CRITICAL, ADVISORY, or PASS results at the next stopping point; renames sentinel to `.surfaced.json` to prevent re-surfacing

**codex-wave-validator-worker.js** (208 lines) is a standalone background process that:
- Parses `--phase`, `--wave`, `--cwd`, `--sentinel`, `--prompt-file` arguments
- Reads the prompt from a temp file (then deletes it) to avoid shell-escaping issues
- Calls `runCodexExec` with 180s timeout via `./codex-exec`
- Extracts Codex text output and parses PASS/CRITICAL/ADVISORY verdict
- Collects `[CRITICAL]` and `[ADVISORY]` prefixed issues into structured array
- Updates the sentinel JSON file with status, summary, issues
- Appends a `task_type: 'wave-validation'` record to `.planning/token-log.jsonl`

### Task 2: codex-plan-reviewer.js + settings.json updates

**codex-plan-reviewer.js** (414 lines) serves two modes:
- **Standalone** (`--phase` or `--help` flag): reviews PLAN.md files in the specified phase, prints verdict, exits
- **SubagentStop hook** (stdin JSON with `hook_event_name: "SubagentStop"`): detects if a planner subagent just ran by checking for PLAN.md files modified within 120 seconds, auto-detects phase, runs Codex review, writes REVIEWS.md, and either blocks (HIGH severity) or passes with advisory context

D-07 enforcement: when HIGH severity issues are found in hook mode, returns `{ decision: "block", reason: "..." }` giving the planner another turn to fix plans before finalizing.

REVIEWS.md filename pattern: `{paddedPhase}-REVIEWS.md` (e.g., `02-REVIEWS.md`) — matches GSD's `has_reviews` detection pattern (files ending with `-REVIEWS.md`).

**~/.claude/settings.json** changes:
- PostToolUse hook array now has 3 entries: gsd-context-monitor.js (10s), codex-token-logger.js (10s), codex-wave-validator.js (10s)
- New `SubagentStop` key added with codex-plan-reviewer.js (300s timeout)
- Stop hook from 02-01 unchanged

## Verification Results

All automated checks pass:

| Check | Result |
|-------|--------|
| `node -c codex-wave-validator.js` | PASS |
| `node -c codex-wave-validator-worker.js` | PASS |
| `node -c codex-plan-reviewer.js` | PASS |
| `node -e "JSON.parse(settings.json)"` | PASS |
| PostToolUse hook count = 3 | PASS |
| SubagentStop hook registered | PASS |
| Stop hook still present | PASS |
| wave-validator timeout = 10 | PASS |
| SubagentStop timeout = 300 | PASS |
| phase-plan-index in wave validator | PASS |
| REVIEWS.md in plan reviewer | PASS |
| decision: block in plan reviewer | PASS |

## Decisions Made

1. **Temp file for prompt passing**: wave validator writes the multi-line validation prompt to `.planning/wave-N-prompt.tmp` before spawning the worker, avoiding any shell-escaping issues with newlines in argv.

2. **Sentinel-before-spawn**: sentinel file is written before `child.unref()` so even if the worker crashes immediately, the sentinel exists and shows `status: "pending"` rather than leaving no trace.

3. **Fail-open in SubagentStop**: if Codex fails in hook mode, plan reviewer exits 0 with advisory context rather than blocking — prevents permanent planner paralysis when Codex is unavailable.

4. **120-second PLAN.md freshness heuristic**: detects planner subagents by checking if any PLAN.md files were modified within the last 120 seconds, without needing to inspect transcript content.

5. **Dual-mode script design**: codex-plan-reviewer.js detects mode via `process.argv.includes('--phase')` — clean and testable without stdin machinery for standalone invocation.

## Deviations from Plan

None — plan executed exactly as written. The `--help` flag handling was added as a minor Rule 2 deviation (missing critical functionality: the verification check `--help 2>&1 | head -1` required the standalone mode to activate for `--help` too).

## Known Stubs

None — all file paths, validation logic, and token logging are fully implemented.

## Self-Check: PASSED

Files exist:
- `~/.claude/hooks/codex-wave-validator.js` — FOUND
- `~/.claude/hooks/codex-wave-validator-worker.js` — FOUND
- `~/.claude/hooks/codex-plan-reviewer.js` — FOUND
- `~/.claude/settings.json` — FOUND (verified valid JSON, 3 PostToolUse hooks, SubagentStop present)
