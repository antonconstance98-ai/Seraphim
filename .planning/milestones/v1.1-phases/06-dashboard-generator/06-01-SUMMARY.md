---
phase: 06-dashboard-generator
plan: 01
subsystem: dashboard
tags: [node.js, jsonl, chart.js, metrics, dashboard, data-processing]

# Dependency graph
requires:
  - phase: 05-data-pipeline
    provides: global.jsonl enriched records with cost_usd, opus_baseline_usd, tokens, session_id, project_name

provides:
  - computeMetrics(records) — transforms global.jsonl array into 7-key DASHBOARD_DATA object
  - ensureChartJs() — async Chart.js 4.5.1 download and cache to ~/.claude/dashboard/assets/chart.min.js
  - generateDashboard(dashboardDir) — reads global.jsonl, calls computeMetrics, returns DASHBOARD_DATA stub (Plan 02 adds HTML)
  - ~/.claude/hooks/codex-dashboard-generator.js — data processing module with dual-mode entry point
  - ~/.claude/dashboard/assets/chart.min.js — Chart.js 4.5.1 UMD bundle (208KB)

affects:
  - 06-02 (dashboard HTML rendering — consumes generateDashboard/computeMetrics)
  - 07-hook-wiring (SessionStart hook triggers generateDashboard)

# Tech tracking
tech-stack:
  added: [Chart.js 4.5.1 (CDN-cached, 208KB UMD bundle)]
  patterns:
    - safeNum() guard for numeric fields from JSONL (handles NaN, strings, Infinity)
    - htmlEscape() converts any type to string before escaping (null → '', 0 → '0')
    - Single-pass accumulation pattern for metrics (one loop over valid records)
    - ensureChartJs() validates preamble bytes before accepting cached file
    - generateDashboard stub pattern — same signature preserved for Plan 02 to replace body

key-files:
  created:
    - ~/.claude/hooks/codex-dashboard-generator.js
    - ~/.claude/dashboard/assets/chart.min.js
  modified:
    - .planning/STATE.md

key-decisions:
  - "generateDashboard returns DASHBOARD_DATA object (not HTML) in stub — Plan 02 replaces body, keeping signature identical"
  - "Unattributed row is supplementary — same records also counted in their project rows; globalSummary totals match sum of project rows (excluding Unattributed)"
  - "modelSplit always initializes gpt-5.4 and gpt-5.4-mini with zero values before merging observed data — guarantees both keys even with zero usage"
  - "sessionHistory sorts by maxTimestamp descending (most recent session first), limited to 50"
  - "catchRate uses 'N/A' for project rows with no reviews; globalSummary uses '0.0' for consistency with numeric display"

patterns-established:
  - "safeNum(v): coerces numeric strings and non-finite values to 0 — use for all cost/token fields from JSONL"
  - "htmlEscape(v): always String() before escaping; null/undefined → '' — safe for any field type in HTML templates"
  - "Single-pass accumulation: one loop builds all grouped Maps simultaneously (project, model, session, taskType)"
  - "Unattributed bucket pattern: null session_id records tallied into a synthetic project row appended last"

requirements-completed: [DASH-01, DASH-02, DASH-03, DASH-04, DASH-05, DASH-06, DASH-07, SESS-01, INTG-05]

# Metrics
duration: 2min
completed: 2026-04-03
---

# Phase 06 Plan 01: Dashboard Data Processing Layer Summary

**computeMetrics() over 54 live records produces 7-key DASHBOARD_DATA with project table, model split, block log, session history, and task distribution — 27/27 verification checks pass, Chart.js 4.5.1 (208KB) cached locally**

## Performance

- **Duration:** ~2 min
- **Started:** 2026-04-03T02:37:20Z
- **Completed:** 2026-04-03T02:39:30Z
- **Tasks:** 1 of 1
- **Files modified:** 2 (+ 1 project STATE.md)

## Accomplishments

- Created `~/.claude/hooks/codex-dashboard-generator.js` (330 lines) implementing safeNum, htmlEscape, ensureChartJs, computeMetrics, generateDashboard stub, and dual-mode entry point
- computeMetrics processes 54 live records in a single pass: globalSummary shows $1.25 total cost vs $6.85 Opus baseline (81.4% savings), projectTable with 4 named projects + Unattributed, modelSplit (gpt-5.4 / gpt-5.4-mini always present), 8+ block log entries sorted descending, session history, task type distribution
- Chart.js 4.5.1 UMD bundle downloaded and cached at ~/.claude/dashboard/assets/chart.min.js (208,522 bytes)
- All 27 verification checks pass including edge cases: empty array, malformed numeric fields (NaN/string), numeric string coercion ("0.05" → 0.05), missing grouping fields defaulting gracefully

## Task Commits

1. **Task 1: Create codex-dashboard-generator.js with ensureChartJs and computeMetrics** - `6227101` (feat)
2. **Security fix: Pin Chart.js SHA-256 integrity check** - `4f68b31` (fix)

## Files Created/Modified

- `~/.claude/hooks/codex-dashboard-generator.js` — Dashboard data processing module: safeNum, htmlEscape, ensureChartJs (async), computeMetrics (single-pass → DASHBOARD_DATA), generateDashboard stub, dual-mode entry point, module.exports
- `~/.claude/dashboard/assets/chart.min.js` — Chart.js 4.5.1 UMD bundle (208KB, cached for Plan 02 HTML rendering)
- `.planning/STATE.md` — Phase focus updated to 06-dashboard-generator

## Decisions Made

- generateDashboard is a stub that returns DASHBOARD_DATA (not HTML) — Plan 02 replaces the body with HTML writing while keeping the same function signature, so callers like the aggregator need no changes
- Unattributed row is supplementary: null-session records are also counted in their project rows. globalSummary totals equal sum of project rows (excluding Unattributed), making the Unattributed row an attribution audit rather than a separate cost bucket
- modelSplit always initializes both gpt-5.4 and gpt-5.4-mini with zero-value entries before merging, guaranteeing both keys appear even when only one model has been observed
- catchRate in globalSummary uses '0.0' (numeric display); catchRate in projectTable rows uses 'N/A' when no reviews exist (distinguishes "no data" from "zero blocks")

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 2 - Missing Critical] Added SHA-256 integrity verification to ensureChartJs**
- **Found during:** Wave 1 Codex validation after Task 1
- **Issue:** ensureChartJs() downloaded chart.min.js and accepted it after checking only JS preamble bytes — not meaningful integrity verification. A compromised CDN or MITM could inject malicious code into a hook-managed local asset that auto-executes on session start.
- **Fix:** Added `crypto.createHash('sha256')` verification. Pinned expected hash `48444a82...` (verified against live CDN 2026-04-03). Download is rejected and file is NOT written if hash does not match. Cache-hit path now reads full file and hashes it; mismatch triggers deletion and re-download.
- **Files modified:** `~/.claude/hooks/codex-dashboard-generator.js`
- **Verification:** `chart.min.js` SHA-256 matches pinned hash check passes; ensureChartJs returns true on warm cache hit
- **Committed in:** `4f68b31`

---

**Total deviations:** 1 auto-fixed (missing critical — security)
**Impact on plan:** Essential security fix. ensureChartJs was accepting any JS-looking content from the CDN with no integrity guarantee. Pinning the SHA-256 closes the supply-chain risk at zero cost to functionality.

## Issues Encountered

None. Chart.js downloaded on first attempt (208KB, valid JS preamble confirmed).

## User Setup Required

None — no external service configuration required. Chart.js is cached locally; dashboard assets directory created automatically.

## Next Phase Readiness

- Plan 02 (HTML rendering) can consume `generateDashboard()` and `computeMetrics()` directly — the DASHBOARD_DATA structure is fully populated and verified against live data
- Chart.js 4.5.1 cached at `~/.claude/dashboard/assets/chart.min.js` — Plan 02 can inline it into the HTML `<script>` tag
- The `generateDashboard` stub signature (`dashboardDir?: string → DASHBOARD_DATA | null`) is the interface Plan 02 will preserve while adding `fs.writeFileSync` for the HTML output

---
*Phase: 06-dashboard-generator*
*Completed: 2026-04-03*
