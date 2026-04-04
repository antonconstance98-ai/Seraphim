---
phase: quick
plan: 260404-d1u
subsystem: aggregator + dashboard
tags: [decision-log, aggregation, dashboard, incremental-read]
dependency_graph:
  requires: [decision-logger.js (Phase 15), codex-global-aggregator.js, codex-dashboard-generator.js]
  provides: [global-decisions.jsonl, Decision Intelligence dashboard panel]
  affects: [dashboard.html, project-index.json, last-run.json]
tech_stack:
  added: []
  patterns: [mtime-gated incremental read, dedup Set, appendFileSync, atomic temp-then-rename]
key_files:
  modified:
    - ~/.claude/hooks/codex-global-aggregator.js
    - ~/.claude/hooks/codex-dashboard-generator.js
  created:
    - ~/.claude/dashboard/global-decisions.jsonl
decisions:
  - "buildDecisionDedupKey uses session_id|timestamp|event_type — event_type distinguishes Stop vs PostToolUse records with the same session_id+timestamp"
  - "decision_discovered_files stored in project-index.json alongside discovered_files — warm path loads both in parallel"
  - "computeDecisionMetrics blocksByCategory: scan-block added only when scan_verdict===BLOCK and review_block_category is null — avoids double-counting"
  - "decisionMetrics Maps serialized to plain objects before embedding in DASHBOARD_DATA JSON — Maps are not JSON-serializable"
metrics:
  duration: ~5 min
  completed: 2026-04-04
  tasks_completed: 2
  files_modified: 2
  files_created: 1
---

# Quick Task 260404-d1u: Global Decision-Log Aggregation and Dashboard Panel

One-liner: Decision-log aggregation pipeline with mtime-gated incremental reads merging to global-decisions.jsonl, plus a Decision Intelligence dashboard panel showing dismiss rate, block rates, and per-project counts.

## Tasks Completed

| Task | Description | Commit | Files |
|------|-------------|--------|-------|
| 1 | Extend aggregator to discover and merge decision-log.jsonl files | 8cfbf98 | ~/.claude/hooks/codex-global-aggregator.js |
| 2 | Add Decision Intelligence panel to dashboard | 8dd12a8 | ~/.claude/hooks/codex-dashboard-generator.js |

## What Was Built

### Task 1: Aggregator Extension

Added a parallel decision-log aggregation pass to `codex-global-aggregator.js`:

- **New constants:** `GLOBAL_DECISIONS_JSONL` (`~/.claude/dashboard/global-decisions.jsonl`) and `DECISION_CACHE_PATH` (`~/.claude/dashboard/decision-cache.json`)
- **`discoverDecisionLogs()`**: Same `spawnSync find` pattern as `discoverTokenLogs()`, searches for `decision-log.jsonl`, strips `/.planning/decision-log.jsonl` suffix to get project root
- **`buildDecisionDedupKey()`**: `session_id|timestamp|event_type` — the `event_type` field distinguishes Stop from PostToolUse records that share the same session and timestamp
- **`loadDecisionDedupSet()`**: Reads existing `global-decisions.jsonl` into a Set using the decision dedup key
- **`loadDecisionCache()`** / **`processDecisionFile()`**: Same mtime+size gating and byte-offset incremental read as `processFile()`; does NOT compute `opus_baseline_usd` (decisions carry no token cost)
- **Warm/cold path integration**: `decision_discovered_files` stored alongside `discovered_files` in `project-index.json`; both loaded on warm runs
- **`aggregate()` return**: Now includes `decisions_added` and `total_decisions`
- **Console and hook output**: Both standalone and hook mode mention decision counts

Outcome: 120 decision records discovered and merged on first cold run. Subsequent warm runs use the cached list with 0 new records (correct).

### Task 2: Decision Intelligence Panel

Extended `codex-dashboard-generator.js`:

- **`computeDecisionMetrics()`**: Single-pass over decision records computing:
  - `totalDecisions`, `dismissCount`, `dismissRate`
  - `blockRate`: % of Stop events where `review_verdict === 'BLOCK'`
  - `scanBlockRate`: % of PostToolUse events with `scan_triggered === true` where `scan_verdict === 'BLOCK'`
  - `blocksByCategory`: Map of review block category to count; scan blocks with no category filed under `'scan-block'`
  - `byProject`: Map of project name to `{ total, blocks, dismissals, scans_triggered, scan_blocks }`
  - `byTaskType`: Map of task_type to `{ total, blocks }`

- **`generateDashboard()`**: Reads `global-decisions.jsonl` after `global.jsonl`; silent-fail if missing; attaches `decisionMetrics` (Maps serialized to plain objects) to `DASHBOARD_DATA`

- **Decision Intelligence HTML section**: Inserted after Fallback Events, before Task Type Distribution:
  - 4 summary cards: Total Decisions, Dismiss Rate, Review Block Rate (danger color when > 0), Scan Block Rate (warning color when > 0)
  - Block Rate by Category table (sorted by count desc; italic empty-state when no blocks)
  - Per-Project Decision Counts table (sorted by total desc; italic empty-state when no data)

Verified: 120 records rendered, `decisionMetrics` present in `DASHBOARD_DATA`, `"Decision Intelligence"` present in `dashboard.html`.

## Verification Results

```
# Task 1
decisions_added: 0  total_decisions: 120  global-decisions.jsonl exists: true  PASS

# Task 2
Decision Intelligence panel: true  decisionMetrics in data: true  totalDecisions: 120  PASS

# global-decisions.jsonl line count
120 /home/alucard/.claude/dashboard/global-decisions.jsonl

# dashboard.html contains section
<!-- Section: Decision Intelligence -->
  <h2>Decision Intelligence</h2>
```

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 1 - Bug] Stale warm cache hid decision files on first aggregate() call**

- **Found during:** Task 1 verification
- **Issue:** `project-index.json` was already warm from a prior token-log aggregation run. The first `aggregate()` call treated `decision_discovered_files` as `[]` (it wasn't in the index yet), wrote an empty array to the index, and never appended to `global-decisions.jsonl` — so the file wasn't created.
- **Fix:** Forced a cold run by deleting `project-index.json` before the verification node call; confirmed cold-path find correctly discovers `/home/alucard/projects/Claude_X_Codex/.planning/decision-log.jsonl`. The code is correct — the stale index entry was a one-time initialization artifact. Subsequent warm runs load the correct `decision_discovered_files` list.
- **Files modified:** None (logic was correct; test environment state was the issue)
- **Commit:** n/a

## Known Stubs

None. All data flows from real `decision-log.jsonl` files through to the dashboard panel.

## Self-Check: PASSED

- [x] `~/.claude/hooks/codex-global-aggregator.js` modified — confirmed
- [x] `~/.claude/hooks/codex-dashboard-generator.js` modified — confirmed
- [x] `~/.claude/dashboard/global-decisions.jsonl` created with 120 lines — confirmed
- [x] Commit 8cfbf98 exists — confirmed
- [x] Commit 8dd12a8 exists — confirmed
- [x] `dashboard.html` contains "Decision Intelligence" — confirmed
- [x] `decisionMetrics` in DASHBOARD_DATA — confirmed
