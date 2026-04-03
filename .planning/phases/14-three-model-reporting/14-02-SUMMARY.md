---
phase: 14-three-model-reporting
plan: "02"
subsystem: dashboard-generator
tags:
  - dashboard
  - minimax
  - chart
  - fallback-events
  - reporting
dependency_graph:
  requires:
    - 14-01 (v2.0 field pass-through in codex-token-logger.js — supplies source, rate_limit_pct fields)
  provides:
    - Three-model dashboard with MiniMax chart series and Fallback Events panel
  affects:
    - ~/.claude/hooks/codex-dashboard-generator.js
tech_stack:
  added: []
  patterns:
    - Pre-initialized modelSplit keys for guaranteed table rows (even with 0 calls)
    - Single-pass fallback event collection alongside existing blockLog pattern
    - Chart.js third dataset with distinct green color (#00e676) for MiniMax series
    - Fallback Events HTML panel with table or empty-state message
    - groupTS weekly aggregation extended for minimaxCost accumulation
key_files:
  created: []
  modified:
    - ~/.claude/hooks/codex-dashboard-generator.js
decisions:
  - modelSplit pre-initialization ensures minimax-m2.7 row appears even when global.jsonl has zero MiniMax records (all 222 current records are gpt-5.4)
  - minimaxCost in timeSeries is MiniMax-only spend; Actual Cost line retains total (Codex + MiniMax combined) — no double-counting
  - Fallback event definition is dual-condition: source==='api-fallback' AND model==='minimax-m2.7' (not just model match)
  - MiniMax color #00e676 matches --text-success CSS variable already defined in dashboard styles
  - Fallback Events panel positioned after Model Split and before Task Type Distribution per HTML section order spec
metrics:
  duration: 2min
  completed_date: "2026-04-03"
  tasks_completed: 2
  files_modified: 1
---

# Phase 14 Plan 02: Three-Model Dashboard (MiniMax Chart Series + Fallback Events Panel) Summary

**One-liner:** Extended codex-dashboard-generator.js with MiniMax as third chart line (#00e676 green), pre-initialized modelSplit row, per-date minimaxCost accumulation, and a Fallback Events health panel showing rate-limit-triggered Codex→MiniMax switches.

## What Was Built

Updated `~/.claude/hooks/codex-dashboard-generator.js` with all changes required to make three-model cost savings immediately visible on the dashboard.

### Task 1: Extend computeMetrics for MiniMax data collection

Five targeted changes to `computeMetrics()` and the client-side `groupTS()`:

1. **modelSplit pre-initialization** — `minimax-m2.7` added as third fixed key with zero values. Ensures the Model Split table always has a MiniMax row even when no MiniMax calls exist in global.jsonl.

2. **byDate minimaxCost accumulation** — `byDate` bucket init gains `minimaxCost: 0`. Conditional accumulation inside the single-pass loop adds to `bucket.minimaxCost` only when `r.model === 'minimax-m2.7'`.

3. **fallbackEntries collection** — `const fallbackEntries = []` declared alongside `blockLogEntries`. Inside the single-pass loop, records matching `source === 'api-fallback' AND model === 'minimax-m2.7'` are pushed with: timestamp, projectName, taskType, tokens (input+output), cost, rateLimitPct.

4. **timeSeries + DASHBOARD_DATA** — timeSeries map extended with `minimaxCost: bucket.minimaxCost`. `fallbackLog` (sorted desc by timestamp) added to the return object.

5. **groupTS weekly aggregation** — weekMap init gains `minimaxCost: 0`; accumulation line `weekMap[key].minimaxCost += ts[i].minimaxCost || 0` added after existing two lines.

### Task 2: HTML template — MiniMax chart series and Fallback Events panel

Three targeted changes to `generateDashboard()`:

1. **fallbackLogHtml builder** — Mirrors blockLogHtml pattern exactly. Empty state: styled italic message. Non-empty: `data-table` with columns Date, Project, Task Type, Tokens, Cost, Rate Limit %.

2. **Fallback Events section in HTML template** — `<section id="fallback-events">` inserted after Model Split (`#models`) and before Task Type Distribution (`#task-types`). Uses `\u2192` for Codex→MiniMax arrow.

3. **MiniMax dataset in buildLine()** — Third dataset object added after Opus Baseline in `datasets` array: label `'MiniMax Cost ($)'`, `borderColor: '#00e676'`, `backgroundColor: 'rgba(0,230,118,0.08)'`, `data: series.map(function(p){ return p.minimaxCost || 0; })`.

## Verification Results

**Task 1 behavioral test (7/7 checks):**
- Syntax valid
- Module loads, exports intact
- modelSplit has minimax-m2.7
- modelSplit minimax calls = 2 (for 2 minimax records)
- fallbackLog has 1 correct entry (api-fallback+minimax only, not api+minimax)
- timeSeries has minimaxCost = 0.0022
- Empty fallbackLog when no fallback events

**Task 2 source checks (7/7 checks):**
- Syntax valid
- Module loads without error
- All source patterns found (fallback-events, fallbackLogHtml, MiniMax Cost, #00e676, rgba(0,230,118,0.08), minimaxCost || 0, data-table, Rate Limit)
- fallback-events positioned before task-types
- MiniMax Cost dataset present
- Empty state message present
- Existing dashboard sections preserved (block-log, session-table, costSavingsChart, projectBarChart)

## Commits

| Task | Commit | Description |
|------|--------|-------------|
| 1 | 0ea50f2 | feat(14-02): extend computeMetrics for MiniMax data collection |
| 2 | 61216d5 | feat(14-02): add MiniMax chart series and Fallback Events panel to dashboard |

## Deviations from Plan

None — plan executed exactly as written. All 5 computeMetrics changes and all 3 HTML template changes applied in order. No architectural changes required.

## Known Stubs

None. The Fallback Events panel and MiniMax chart series are fully wired. The MiniMax series will show a flat zero line until real `api-fallback` + `minimax-m2.7` records appear in global.jsonl — this is correct behavior, not a stub.

## Self-Check: PASSED

- FOUND: .planning/phases/14-three-model-reporting/14-02-SUMMARY.md
- FOUND: ~/.claude/hooks/codex-dashboard-generator.js
- FOUND: commit 0ea50f2 (Task 1)
- FOUND: commit 61216d5 (Task 2)
