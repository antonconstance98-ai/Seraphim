---
phase: 05-data-pipeline
plan: 03
subsystem: infra
tags: [node.js, jsonl, aggregator, caching, mtime, discovery, performance]

# Dependency graph
requires:
  - phase: 05-data-pipeline-plan-02
    provides: codex-global-aggregator.js with discoverTokenLogs() and aggregate()
provides:
  - TTL-gated discovery cache in codex-global-aggregator.js — warm runs skip spawnSync find
  - discovered_files array persisted in project-index.json for warm-run lookup
  - isCacheWarm() and loadDiscoveryCache() helpers in codex-global-aggregator.js
affects: [06-dashboard-generator, 07-session-hook-wiring]

# Tech tracking
tech-stack:
  added: []
  patterns:
    - "TTL-gated cache pattern: check file mtime vs constant before running expensive subprocess"
    - "Warm/cold branch in aggregate(): wasWarm flag captured before any writes"
    - "Carry-forward pattern: warm run reads existing discovered_files rather than overwriting with filtered subset"

key-files:
  created: []
  modified:
    - "~/.claude/hooks/codex-global-aggregator.js — added DISCOVERY_CACHE_TTL_MS, isCacheWarm(), loadDiscoveryCache(), warm/cold discovery branch, discovered_files persistence"
    - ".planning/phases/05-data-pipeline/05-VERIFICATION.md — added to repo (verifier output from plan 02)"

key-decisions:
  - "TTL of 1 hour chosen: covers all normal session cadences, cold starts always re-discover"
  - "wasWarm captured before any writes: prevents race between warm check and index write"
  - "Carry-forward on warm run: write existing discovered_files, not filtered subset — avoids shrinking cache on unmounted drive"
  - "discoverTokenLogs() left completely unchanged: only aggregate() caller changes"

patterns-established:
  - "TTL-gated subprocess cache: check index file mtime before spawning child_process"
  - "discovered_files top-level key in project-index.json: backward-compatible schema extension"

requirements-completed: [PIPE-03]

# Metrics
duration: 3min
completed: 2026-04-03
---

# Phase 5 Plan 03: Discovery Cache (Gap Closure) Summary

**Mtime-gated discovery cache added to codex-global-aggregator.js — warm no-op runs complete in 2ms (ROADMAP target: <5ms) by skipping 4x spawnSync find child processes**

## Performance

- **Duration:** 3 min
- **Started:** 2026-04-03T00:46:13Z
- **Completed:** 2026-04-03T00:49:51Z
- **Tasks:** 1
- **Files modified:** 1 (system file outside repo)

## Accomplishments

- Warm no-op run now completes in 2ms (down from 151ms) — closes ROADMAP Success Criterion 3
- `discovered_files` array persisted in `project-index.json` on cold runs; read back on warm runs
- `isCacheWarm()` checks `project-index.json` mtime vs 1-hour TTL before any subprocess spawn
- `discoverTokenLogs()` and all existing contracts left completely unchanged
- Cold run still discovers all 4 projects and writes `discovered_files` with correct entries
- Hook mode (`additionalContext` JSON) unchanged and verified

## Task Commits

Each task was committed atomically:

1. **Task 1: Add mtime-gated discovery cache to codex-global-aggregator.js** - `e4c1be0` (feat)

**Plan metadata:** (included in final docs commit)

## Files Created/Modified

- `~/.claude/hooks/codex-global-aggregator.js` — Added `DISCOVERY_CACHE_TTL_MS` constant, `isCacheWarm()`, `loadDiscoveryCache()`, warm/cold branch in `aggregate()` step 3, `discovered_files` persistence in step 7 (system file, outside repo — not tracked in project git)
- `.planning/phases/05-data-pipeline/05-VERIFICATION.md` — Added to repo (was previously untracked; documents the gap source)

## Decisions Made

- **1-hour TTL:** Covers all normal session cadences while ensuring stale discovery doesn't persist more than a session. Cold starts (no `project-index.json`) always run full `find`.
- **`wasWarm` captured before writes:** The warm/cold flag is set at the start of `aggregate()` before any state file writes. This prevents the index write in step 7 from interfering with the branch decision.
- **Carry-forward on warm runs:** The warm path reads the existing `discovered_files` from disk and preserves it as-is, rather than writing back the filtered subset. This prevents cache shrinkage if a drive is temporarily unmounted during a warm run.
- **`discoverTokenLogs()` left unchanged:** Only `aggregate()` changes which path calls it. The function signature, behavior, and filtering logic are preserved exactly.

## Deviations from Plan

None — plan executed exactly as written. All 5 edit steps applied as specified. Warm run elapsed_ms = 2ms (target: <5ms). All 9 acceptance criteria passed on first attempt.

## Issues Encountered

**Hook mode / TTY detection:** Running `node codex-global-aggregator.js` directly in a Bash subshell (no TTY) routes to hook mode, which waits for stdin. Used `< /dev/null` to close stdin immediately and trigger hook mode for the cold-run verification step. This is expected behavior — the file is designed for dual-mode operation.

## User Setup Required

None — no external service configuration required. The cache is transparent: first run after any `project-index.json` deletion or expiry runs full discovery automatically.

## Next Phase Readiness

- Phase 5 (data-pipeline) is now fully complete: all 4 PIPE requirements satisfied, ROADMAP Success Criterion 3 met (2ms warm run)
- Phase 6 (dashboard generator) can `require('./codex-global-aggregator')` and call `aggregate()` — the interface is stable
- `global.jsonl` has 53 records from 4 projects, ready for dashboard rendering

## Self-Check: PASSED

- FOUND: 05-03-SUMMARY.md
- FOUND: commit e4c1be0
- FOUND: hook syntax OK (node --check passes)
- FOUND: DISCOVERY_CACHE_TTL_MS in hook file
- FOUND: isCacheWarm in hook file
- FOUND: wasWarm in hook file
- FOUND: discovered_files in hook file

---
*Phase: 05-data-pipeline*
*Completed: 2026-04-03*
