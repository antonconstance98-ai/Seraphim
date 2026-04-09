---
phase: 14-three-model-reporting
plan: 01
subsystem: reporting
tags: [token-logging, cost-reporting, minimax, three-model, fallback-tracking]

# Dependency graph
requires:
  - phase: 13-codex-execution-pipeline
    provides: codex-handoff.js with MiniMax fallback emitting source='api-fallback' and model='minimax-m2.7'
  - phase: 08-minimax-foundation
    provides: minimax-m2.7 pricing in codex-pricing.js, computeOpusCost export
provides:
  - v2.0 optional field pass-through in codex-token-logger.js (dual_review, review_round, round_model, fallback_from, compression)
  - Three-model cost breakdown in codex-cost-reporter.js (Codex + MiniMax + Opus baseline counterfactual)
  - Fallback event detection and reporting (source==='api-fallback' AND model==='minimax-m2.7')
  - Model-neutral Summary labels ("Actual Cost", "Total Calls")
  - Extended generateReport return object with fallbackCount and fallbackCost
affects: [14-02-dashboard-update, any future plan consuming generateReport return object]

# Tech tracking
tech-stack:
  added: []
  patterns:
    - "Conditional field mutation after object literal for backward-compatible v2.0 field extension"
    - "Dual-condition fallback detection: source==='api-fallback' AND model==='minimax-m2.7'"
    - "Counterfactual baseline labeling: 'Opus Baseline (what this would have cost)' to distinguish computed from recorded costs"

key-files:
  created: []
  modified:
    - "~/.claude/hooks/codex-token-logger.js"
    - "~/.claude/hooks/codex-cost-reporter.js"

key-decisions:
  - "v2.0 fields added as post-literal mutations (not inside object) to keep record schema visually clean and preserve !== undefined semantics"
  - "Fallback detection uses BOTH source==='api-fallback' AND model==='minimax-m2.7' -- dual condition prevents false positives from future API-fallback sources"
  - "fallbackCost = sum of cost_usd for matching records (full MiniMax cost, not incremental) -- Codex produced zero cost on these calls since it failed"
  - "Opus baseline labeled as counterfactual ('what this would have cost') to distinguish from recorded costs in the token log"
  - "Three-Model Breakdown placed after Model Comparison (not before) to maintain existing report section order"

patterns-established:
  - "Pattern: !== undefined guard for optional field pass-through -- preserves false and 0 values that truthiness checks would drop"
  - "Pattern: Section ordering in cost report -- Summary, Breakdown by Task Type, Model Comparison, Three-Model Breakdown, Fallback Events, Review Activity"

requirements-completed: [D-03, D-04, D-05, D-06, D-07]

# Metrics
duration: 2min
completed: 2026-04-03
---

# Phase 14 Plan 01: Three-Model Reporting Summary

**Token logger v2.0 field pass-through (5 optional fields via !== undefined) and cost reporter three-model breakdown with Codex/MiniMax/Opus-baseline sections and fallback event counting**

## Performance

- **Duration:** 2 min
- **Started:** 2026-04-03T21:44:29Z
- **Completed:** 2026-04-03T21:46:51Z
- **Tasks:** 2
- **Files modified:** 2 (both outside repo at ~/.claude/hooks/)

## Accomplishments

- codex-token-logger.js now conditionally forwards five v2.0 metadata fields (dual_review, review_round, round_model, fallback_from, compression) from [CODEX_RESULT] markers to JSONL records, using !== undefined checks so false and 0 pass through correctly
- codex-cost-reporter.js now generates a Three-Model Breakdown section separating Codex execution cost, MiniMax analysis cost, and Opus baseline (labeled as counterfactual), replacing the misleading "Codex-only" Summary labels
- Fallback event detection added: records matching source==='api-fallback' AND model==='minimax-m2.7' are counted and their cost summed, reported in both the Markdown report and hook-mode advisory context
- generateReport return object extended with fallbackCount and fallbackCost (backward compatible -- no existing callers read these fields)

## Task Commits

Each task was committed atomically:

1. **Task 1: Add v2.0 field pass-through to codex-token-logger.js** - `4c810f6` (feat)
2. **Task 2: Update codex-cost-reporter.js for three-model breakdown and fallback tracking** - `5c239f8` (feat)

**Plan metadata:** see final commit below

## Files Created/Modified

- `~/.claude/hooks/codex-token-logger.js` - Added 5 conditional v2.0 field mutations after record object literal
- `~/.claude/hooks/codex-cost-reporter.js` - Renamed Summary labels, added fallbackEvents array, three-model breakdown section, fallback events section, extended return object, added fallback advisory context

## Decisions Made

- v2.0 fields are added as post-literal mutations (not inside the object literal) to keep the record schema visually distinct from v1.0 fields and to clearly signal their optional/conditional nature
- Fallback detection requires BOTH conditions (source + model) to be a future-proof guard against other api-fallback sources being added later
- fallbackCost represents the full MiniMax cost (not incremental above Codex) because Codex produced zero cost on these calls (it failed before billing)
- Opus baseline is labeled "what this would have cost" to make the counterfactual nature explicit -- operators should not confuse this with recorded Opus API spend

## Deviations from Plan

None - plan executed exactly as written. All 6 changes applied in the specified order. All verification checks passed.

## Issues Encountered

Minor: Hook files at ~/.claude/hooks/ are outside the project git repository, so task commits were made as empty commits with descriptive messages noting the external file path. This matches the pattern established in Phase 13.

## User Setup Required

None - no external service configuration required. Hook files are already installed and active.

## Next Phase Readiness

- Plan 14-02 (dashboard update for three-model metrics) can now consume the extended generateReport return object with fallbackCount and fallbackCost
- The token-log.jsonl schema is forward-compatible: existing records parse without errors; new records from MiniMax fallback executions will carry the source and model fields needed for three-model breakdown
- Both files pass node -c syntax check and all acceptance criteria verified

---
*Phase: 14-three-model-reporting*
*Completed: 2026-04-03*
