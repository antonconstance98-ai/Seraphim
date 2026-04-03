---
phase: 05-data-pipeline
plan: 02
subsystem: data-pipeline
tags: [node.js, jsonl, aggregator, dedup, incremental-reads, mtime-cache, atomic-writes]

# Dependency graph
requires:
  - phase: 05-01
    provides: computeOpusCost from codex-pricing.js — used for opus_baseline_usd enrichment
provides:
  - "~/.claude/hooks/codex-global-aggregator.js — global JSONL aggregator with discovery, dedup, incremental reads, enrichment"
  - "~/.claude/dashboard/global.jsonl — merged, deduplicated records from all projects"
  - "~/.claude/dashboard/project-index.json — discovery manifest with per-project record counts"
  - "~/.claude/dashboard/last-run.json — idempotency guard with run metadata"
  - "~/.claude/dashboard/cache.json — incremental read state (mtime + byte offset per source file)"
  - "~/.claude/dashboard/config.json — user-configurable discovery roots and depth settings"
affects: [06-dashboard-generator, 07-hook-wiring]

# Tech tracking
tech-stack:
  added: []
  patterns:
    - "mtime + size comparison for incremental file reads (fast path: skip unchanged files)"
    - "buildDedupKey: session_id|timestamp — handles null session_id as 'null|timestamp' string"
    - "atomicWriteJSON: write-to-temp then renameSync — prevents corruption from concurrent sessions"
    - "execSync find with -not -path exclusions for project discovery"
    - "Dual-mode entry point: process.stdin.isTTY for standalone vs hook mode"

key-files:
  created:
    - "~/.claude/hooks/codex-global-aggregator.js — 374-line global aggregator (system file, outside repo)"
    - "~/.claude/dashboard/global.jsonl — 47 deduplicated records from 4 projects"
    - "~/.claude/dashboard/project-index.json — 4-project discovery manifest"
    - "~/.claude/dashboard/last-run.json — run metadata with records_added, total_records, elapsed_ms"
    - "~/.claude/dashboard/cache.json — mtime + byte_offset per source file"
    - "~/.claude/dashboard/config.json — extra_roots: [], max_depth: 5"
  modified:
    - ".planning/STATE.md — position updated"

key-decisions:
  - "Aggregator already existed from prior work session — verified all acceptance criteria rather than recreating"
  - "4 null session_id records preserved in global.jsonl — PIPE-04 requirement met without code changes"
  - "Idempotency confirmed via mtime + size comparison: elapsed_ms=155 on no-op run (no new I/O beyond stat())"

patterns-established:
  - "Global aggregation pattern: discover → dedup Set → incremental read → enrich → append → atomic state write"
  - "project_name derived from path.basename(path.dirname(path.dirname(filePath))) for /.planning/token-log.jsonl layout"
  - "opus_baseline_usd computed at aggregation time (not at logging time) via computeOpusCost(rec.tokens)"

requirements-completed: [PIPE-01, PIPE-02, PIPE-03, PIPE-04]

# Metrics
duration: 3min
completed: 2026-04-03
---

# Phase 05 Plan 02: Global Aggregator Summary

**Multi-project JSONL aggregator with mtime-gated incremental reads, dedup via session_id|timestamp key, null session_id preservation, and opus_baseline_usd enrichment from codex-pricing.js**

## Performance

- **Duration:** ~3 min
- **Started:** 2026-04-03T00:19:07Z
- **Completed:** 2026-04-03T00:21:31Z
- **Tasks:** 2 completed
- **Files modified:** 1 (STATE.md in repo; aggregator and dashboard files are system-level outside repo)

## Accomplishments

- Global aggregator (`codex-global-aggregator.js`, 374 lines) discovers and merges token-log.jsonl from all known projects into a single deduplicated `global.jsonl`
- 47 records across 4 projects (Claude_X_Codex, The-Crucible, vibe-edit, app) with all enrichment fields present (project_name, project_path, opus_baseline_usd as number)
- 4 null session_id records preserved in global.jsonl — PIPE-04 met (not dropped, tagged as unattributed by key "null|timestamp")
- Idempotency verified across 3 consecutive runs: records_added=0, line count unchanged, elapsed_ms=155 (fast path working)
- All state files written atomically via write-then-rename pattern

## Task Commits

1. **Task 1: Create codex-global-aggregator.js** - `1ed2d20` (feat)
2. **Task 2: Verify idempotency** - `1855db5` (feat)

## Files Created/Modified

System files (outside git repo, in `~/.claude/`):
- `~/.claude/hooks/codex-global-aggregator.js` — 374-line aggregator; exports `aggregate()` for require() by Phase 6
- `~/.claude/dashboard/global.jsonl` — 47 deduplicated records from 4 projects
- `~/.claude/dashboard/project-index.json` — per-project metadata (project_name, last_seen, record_count)
- `~/.claude/dashboard/last-run.json` — run metadata (timestamp, projects_scanned, records_added, total_records, elapsed_ms)
- `~/.claude/dashboard/cache.json` — mtime_ms, size, byte_offset per source file
- `~/.claude/dashboard/config.json` — extra_roots: [], max_depth: 5

Planning files (in git repo):
- `.planning/STATE.md` — position updated to 05-02 complete

## Decisions Made

- The aggregator file already existed from a prior work session (git timestamp Apr 2 19:17, before this plan execution). Rather than recreating it, the plan acceptance criteria were verified against the existing file — all checks passed.
- Idempotency relies on both the mtime+size fast path AND the dedup Set. The cache fast path skips file reads; the dedup Set catches any records that slip through if cache is stale.
- `total_records` in last-run.json is `initialSize + recordsAdded` (dedup Set size after processing), which correctly matches `wc -l global.jsonl`.

## Deviations from Plan

None — plan executed as written. The aggregator file existed and passed all acceptance criteria without modification.

## Issues Encountered

None. The aggregator was already complete and correctly implemented. Both verification passes (Task 1 full verification, Task 2 idempotency triple-check) passed on first attempt.

## User Setup Required

None — no external service configuration required. All files are written to `~/.claude/dashboard/` automatically.

## Next Phase Readiness

- `global.jsonl` is ready as the root data source for Phase 6 (dashboard generator)
- `aggregate()` is exported and can be required by the dashboard generator via `require('./codex-global-aggregator')`
- All state files are consistent and atomic-write safe
- Discovery roots configurable via `~/.claude/dashboard/config.json` (extra_roots array)
- Phase 7 (hook wiring) can register the aggregator as a SessionStart hook; standalone mode is verified working

---
*Phase: 05-data-pipeline*
*Completed: 2026-04-03*
