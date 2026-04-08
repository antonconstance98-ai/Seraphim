---
phase: 03-six-phase-pipeline-and-profile-management
plan: 01
subsystem: dispatch-cli-markers-banner
tags: [dispatch, cli, markers, banner, PIPE-07]
dependency_graph:
  requires: ["01-plugin-scaffold", "02-model-executors"]
  provides: ["dispatch-cli-entry", "lib/markers", "lib/banner"]
  affects: ["all-phase-commands", "feedback-loops", "terminal-output"]
tech_stack:
  added: []
  patterns:
    - "require.main === module CLI entry pattern"
    - "HTML comment marker round-trip (PIPE-07)"
    - "Unicode box-drawing banner (D-06, D-07)"
key_files:
  created:
    - ~/.claude/plugins/seraphim/lib/markers.js
    - ~/.claude/plugins/seraphim/lib/banner.js
  modified:
    - ~/.claude/plugins/seraphim/executors/dispatch.js
decisions:
  - "Plugin prerequisite files (phases 01-02) were missing from disk — reconstructed in-line as Rule 3 (blocking issue). Files referenced /home/alucard/ in old SUMMARYs but user is alucardmessangeroflight — directory never existed."
  - "EXECUTOR_MAP in CLI is a static lookup in dispatch.js rather than models.json executorFile field — avoids modifying the config schema, keeps mapping colocated with the code that uses it"
  - "token-logger.js is a stdin/stdout hook, not a library — dispatch CLI catches require() failure and skips logging silently, per plan spec"
  - "renderWingBanner returns string (not printWingBanner) — plan action specified export name as renderWingBanner"
metrics:
  duration_min: 12
  completed_date: "2026-04-08"
  tasks_completed: 1
  tasks_total: 1
  files_created: 2
  files_modified: 1
---

# Phase 03 Plan 01: Dispatch CLI, Markers, and Banner Summary

**One-liner:** CLI entry point appended to dispatch.js for external model invocation, SERAPHIM HTML comment marker parser/emitter (PIPE-07), and six-wing Unicode terminal banner renderer.

## What Was Built

### executors/dispatch.js — CLI entry point

`require.main === module` block appended to the existing dispatch module. Parses `--phase`, `--prompt-file`, `--project-root`, and `--output-file` flags. Calls `config.read()` and `resolveExecutorId()`, routes to the correct `*-exec.js` file via a static `EXECUTOR_MAP`, reads the prompt file, calls `executor.execute()`, and writes output atomically (tmp + rename). Prints usage to stderr and exits 1 when required flags are missing or any step fails. Native models (claude-opus-4-6, claude-sonnet-4-6) print a clear error and exit 1.

Existing exports (`resolveExecutorId`, `resolveProfile`) are unchanged.

### lib/markers.js — SERAPHIM HTML comment marker parser/emitter (PIPE-07)

- `parseMarkers(content)` — regex `/<!--\s*SERAPHIM:(\w+)\s+([\s\S]*?)-->/g` with nested `/(\w+)="([^"]*)"/g` attr parsing. Returns array of `{ type, ...attrs }`.
- `emitMarker(type, attrs)` — builds `<!-- SERAPHIM:TYPE key="val" -->` string from attrs object. Quotes `"` chars in values as `&quot;`.
- `emitPhaseStart(phase, model, profile)` — PHASE_START with timestamp
- `emitPhaseComplete(attrs)` — PHASE_COMPLETE with timestamp merged in
- `emitApproach(id, verdict, reason)` — APPROACH marker for Judge phase
- `emitBlueprint(projectType, phase, taskCount)` — BLUEPRINT marker for Architect phase
- `emitTrackFailed(model, error)` — TRACK_FAILED marker (Pitfall 6)

All functions are pure — no side effects, no file I/O.

### lib/banner.js — Six-wing terminal banner

- `WING_MAP` — phase slug to wing label mapping for all six wings
- `renderWingBanner(phase, opts)` — returns formatted string using `═` (heavy) and `─` (light) Unicode box-drawing characters. Format matches D-06 spec with model, profile, phase ID, status, and cost rows. No emojis per project conventions.

## Verification Results

All automated checks passed:

```
marker: <!-- SERAPHIM:PHASE_START phase="judge" model="gemini-3-flash" ... -->
parsed: [{"type":"PHASE_START","phase":"judge","model":"gemini-3-flash",...}]
Banner includes "Wing III" and "Judge": PASS
dispatch --phase (missing args): prints usage with --phase flag: PASS
ALL CHECKS PASSED
```

Acceptance criteria (6/6 grep counts, 3/3 module loads): ALL GREEN

## Tasks Completed

| Task | Name | Commit | Files |
|------|------|--------|-------|
| 1 | Dispatch CLI entry point and marker/banner libraries | 1acad38 | executors/dispatch.js, lib/markers.js, lib/banner.js |

## Deviations from Plan

### Auto-fixed Issues (Rule 3 — blocking)

**1. [Rule 3 - Blocking] Reconstructed missing plugin prerequisite files**
- **Found during:** Task 1 setup — phases 01 and 02 plugin files not present on disk
- **Issue:** `~/.claude/plugins/seraphim/` directory did not exist. Phase 01-02 SUMMARYs referenced `/home/alucard/` (non-existent user); current user is `alucardmessangeroflight`. Plugin was never created at the correct path.
- **Fix:** Created full plugin directory structure and all prerequisite files from scratch using phase 01-02 SUMMARY specifications: plugin.json, models.json, profiles.json, config.js, phase-state.js, dispatch.js (base), pricing.js, token-logger.js.
- **Files created:** 8 prerequisite files + the 2 new plan deliverables
- **Commit:** 1acad38 (combined into one initial commit since plugin git repo was being initialized)

## Known Stubs

None. All three modules (dispatch CLI, markers, banner) are fully wired with no placeholder values, hardcoded empty returns, or TODO stubs in the execution path.

## Self-Check: PASSED

| Item | Status |
|------|--------|
| ~/.claude/plugins/seraphim/executors/dispatch.js | FOUND |
| ~/.claude/plugins/seraphim/lib/markers.js | FOUND |
| ~/.claude/plugins/seraphim/lib/banner.js | FOUND |
| require.main === module in dispatch.js | 1 match |
| parseMarkers in markers.js | 2 matches |
| emitPhaseStart in markers.js | 2 matches |
| emitPhaseComplete in markers.js | 2 matches |
| renderWingBanner in banner.js | 3 matches |
| WING_MAP in banner.js | 3 matches |
| Marker round-trip test | PASSED |
| Banner Wing III render test | PASSED |
| dispatch.js --phase flag in help | PASSED |
| commit 1acad38 | FOUND |
