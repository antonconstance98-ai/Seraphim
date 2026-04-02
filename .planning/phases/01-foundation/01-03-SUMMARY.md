---
phase: 01-foundation
plan: "03"
subsystem: hooks
tags: [gap-closure, codex-router, token-logger, pretooluse, posttooluse, settings]
dependency_graph:
  requires:
    - 01-01 (codex-exec.js shared wrapper)
    - 01-02 (codex-token-logger.js implementation)
  provides:
    - codex-router.js (PreToolUse advisory hook)
    - hook registration in ~/.claude/settings.json
  affects:
    - All future Write/Edit tool calls (PreToolUse advisory fires)
    - All future Bash/Edit/Write/MultiEdit/Agent/Task calls (PostToolUse token logger fires)
tech_stack:
  added: []
  patterns:
    - PreToolUse advisory hook pattern (additionalContext, exit 0 always)
    - Stdin timeout guard (5s for PreToolUse, 10s for PostToolUse)
    - Project-scope config opt-in pattern (reads .claude/settings.json routing_enabled)
key_files:
  created:
    - /home/alucard/.claude/hooks/codex-router.js
  modified:
    - /home/alucard/.claude/settings.json
decisions:
  - "Advisory-only routing in Phase 1: codex-router.js injects context, Opus decides whether to delegate — universal auto-routing via keyword analysis deferred (fragile)"
  - "Router does not require codex-exec.js: hook advises only, Opus invokes codex-exec.js when it decides to delegate — cleaner separation"
  - "Comment reword: plan verification check was `!includes('permissionDecision')` — advisory comment containing that string caused false failure; rewrote to avoid string match"
metrics:
  duration: "3 minutes"
  completed: "2026-04-02T18:17:00Z"
  tasks_completed: 2
  files_created: 1
  files_modified: 1
requirements_closed:
  - FNDTN-03
  - ROUT-03
  - ROUT-04
  - TRCK-01
  - TRCK-02
---

# Phase 01 Plan 03: Gap Closure (Routing Hook + Hook Registration) Summary

**One-liner:** PreToolUse advisory routing hook created and both Codex hooks registered in user-scope settings, closing all three Phase 1 gaps.

## What Was Built

This plan closed the three gaps identified by Phase 1 verification:

**Gap 1 — codex-router.js missing (FNDTN-03):** Created the PreToolUse advisory hook at `~/.claude/hooks/codex-router.js`. The hook reads project-scope `.claude/settings.json` for `routing_enabled` (D-07 opt-in), `preferred_model` (ROUT-04), `fallback_on_error` (ROUT-03), and `attribution_enabled` (D-08). When routing is enabled, it injects an `additionalContext` advisory telling Claude to consider delegating Write/Edit operations to Codex. Hook never blocks — always exits 0.

**Gap 2 — codex-token-logger.js unregistered (TRCK-01, TRCK-02):** Registered the existing (but orphaned) `codex-token-logger.js` as the second hook in the PostToolUse `Bash|Edit|Write|MultiEdit|Agent|Task` matcher with `timeout: 10`. Token logging will now fire on every qualifying tool call.

**Gap 3 — ROUT-03/ROUT-04 config-only (no runtime enforcement):** Resolved by codex-router.js reading and acting on `fallback_on_error` and `preferred_model` from project config at runtime. The advisory message communicates both values to Claude at every Write/Edit intercept.

## Tasks Completed

| Task | Name | Commit | Files |
|------|------|--------|-------|
| 1 | Create codex-router.js PreToolUse routing hook | 2fc23c4 | `~/.claude/hooks/codex-router.js` (102 lines, created) |
| 2 | Register both hooks in user-scope settings | 5829696 | `~/.claude/settings.json` (modified, additive only) |

## Artifacts Produced

**~/.claude/hooks/codex-router.js** (102 lines)
- PreToolUse advisory hook
- Reads `{cwd}/.claude/settings.json` for routing config
- Checks `routing_enabled !== true` → exits silently (D-07 opt-in)
- Extracts `preferred_model`, `api_model`, `fallback_on_error`, `attribution_enabled`, `timeout_seconds`
- Outputs `additionalContext` advisory JSON when routing is enabled
- Never denies/blocks tool execution (advisory pattern)
- Stdin timeout: 5s (matches PreToolUse registration timeout)

**~/.claude/settings.json** (additive changes only)
- PreToolUse `Write|Edit` matcher: now has 3 hooks (guard → gsd-prompt → codex-router, all timeout 5)
- PostToolUse `Bash|Edit|Write|MultiEdit|Agent|Task` matcher: now has 2 hooks (gsd-context-monitor → codex-token-logger, both timeout 10)
- All existing hooks, permissions, plugins, statusLine, preferences preserved exactly

## Hook Registration State After This Plan

```
PreToolUse (Write|Edit):
  1. claude-settings-guard.js    timeout=5  [existing - security guard]
  2. gsd-prompt-guard.js         timeout=5  [existing - workflow guard]
  3. codex-router.js             timeout=5  [NEW - advisory routing]

PostToolUse (Bash|Edit|Write|MultiEdit|Agent|Task):
  1. gsd-context-monitor.js      timeout=10 [existing - GSD context]
  2. codex-token-logger.js       timeout=10 [NEW - token tracking]

SessionStart:
  1. gsd-check-update.js                    [existing - GSD updates]
```

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 1 - Bug] Advisory comment contained verification-failing string**
- **Found during:** Task 1 — initial verification check
- **Issue:** Plan's verify command included `!src.includes('permissionDecision')` but the original comment read "additionalContext pattern, NOT permissionDecision deny" — the string appeared in the comment, causing the check to fail
- **Fix:** Rewrote comment to "additionalContext pattern only (advisory, not blocking)" — removes the test string from source while preserving the semantic intent
- **Files modified:** `~/.claude/hooks/codex-router.js` (comment on line 87)
- **Commit:** 2fc23c4 (included in same task commit)

### Out of Scope (Not Fixed)

None identified.

## Requirements Closed by This Plan

| Requirement | Status Before | Status After |
|-------------|--------------|--------------|
| FNDTN-03 | Blocked — hook missing | SATISFIED — codex-router.js created |
| ROUT-03 | Blocked — config only, no enforcement | SATISFIED — router reads and communicates fallback_on_error at runtime |
| ROUT-04 | Blocked — config only, no enforcement | SATISFIED — router reads and communicates preferred_model at runtime |
| TRCK-01 | Blocked — hook unregistered | SATISFIED — codex-token-logger.js registered in PostToolUse |
| TRCK-02 | Blocked — hook unregistered | SATISFIED — same as TRCK-01 |

## Known Stubs

None. Both hooks are fully wired. The router will only produce meaningful advisory output once `routing_enabled` is set to `true` in a project's `.claude/settings.json` — this is by design (D-07 opt-in). The project-scope config currently has `routing_enabled: false`, which is the correct default.

## Self-Check: PASSED

| Item | Status |
|------|--------|
| `~/.claude/hooks/codex-router.js` exists | FOUND |
| `~/.claude/settings.json` exists | FOUND |
| `01-03-SUMMARY.md` exists | FOUND |
| Commit 2fc23c4 (Task 1) | FOUND |
| Commit 5829696 (Task 2) | FOUND |
