---
phase: 08-minimax-foundation
plan: 01
subsystem: pricing
tags: [pricing, migration, minimax, opus, dashboard, jsonl]

# Dependency graph
requires:
  - phase: 07-charts-hook-integration
    provides: dashboard HTML generation via codex-dashboard-generator.js
  - phase: 05-data-pipeline
    provides: codex-pricing.js module, global.jsonl schema, computeOpusCost function
provides:
  - Corrected Opus 4.6 pricing ($5/$25) in codex-pricing.js
  - MiniMax M-2.7 pricing entry (input:0.30, cached:0.06, output:1.20) in codex-pricing.js
  - All 215 global.jsonl records recalculated with corrected opus_baseline_usd
  - Idempotent migration script at ~/.claude/hooks/migrate-opus-pricing.js
  - Dashboard regenerated with corrected savings percentages
affects:
  - 08-02 (minimax-exec.js will consume minimax-m2.7 pricing)
  - 08-03 (dashboard will show corrected three-model savings metrics)
  - All future plans that read opus_baseline_usd from global.jsonl

# Tech tracking
tech-stack:
  added: []
  patterns:
    - Atomic JSONL rewrite via write-to-tmp-then-renameSync (matches dashboard pattern)
    - Idempotent migration: deterministic recalculation from stored token counts
    - Centralized pricing module as single source of truth for all model rates

key-files:
  created:
    - ~/.claude/hooks/migrate-opus-pricing.js
  modified:
    - ~/.claude/hooks/codex-pricing.js

key-decisions:
  - "Opus 4.6 pricing is $5/$25 per 1M tokens (not $15/$75 which was Opus 4.1 -- a 3x error)"
  - "MiniMax M-2.7 pricing added to CODEX_PRICING not a separate constant -- consistent with existing pattern"
  - "Migration rewrites global.jsonl only (per-project token-log.jsonl never contains opus_baseline_usd)"
  - "Migration is idempotent by design: recalculates deterministically from stored tokens, not from old stored values"

patterns-established:
  - "JSONL migration: read-all, transform-per-line, preserve blanks/malformed, atomic rename, regenerate dashboard"

requirements-completed:
  - D-07
  - D-08
  - D-09

# Metrics
duration: 8min
completed: 2026-04-03
---

# Phase 08 Plan 01: MiniMax Foundation -- Pricing Correction Summary

**Corrected 3x Opus pricing error ($15/$75 Opus 4.1 -> $5/$25 Opus 4.6), added MiniMax M-2.7 pricing, and recalculated all 215 global.jsonl records; savings percentages now reflect real cost data**

## Performance

- **Duration:** 8 min
- **Started:** 2026-04-03T18:03:00Z
- **Completed:** 2026-04-03T18:11:23Z
- **Tasks:** 2
- **Files modified:** 2 (both outside repo: ~/.claude/hooks/)

## Accomplishments

- Fixed Opus pricing from $15/$75 (Opus 4.1 rates, wrong) to $5/$25 (Opus 4.6 rates, correct) in codex-pricing.js
- Added MiniMax M-2.7 pricing to CODEX_PRICING: input $0.30, cached_input $0.06, output $1.20 per 1M tokens
- Created and ran idempotent migration script that corrected all 215 historical global.jsonl records
- Old opus_baseline total: $173.74 (at wrong $15/$75). New total: $57.91 (at correct $5/$25) -- 3x reduction
- Dashboard regenerated with corrected data; dashboard.html reflects accurate savings percentages

## Task Commits

Hook files live outside the git repo (~/.claude/hooks/), so per-task commits are not possible.
Changes tracked in final metadata commit only (project pattern established in Phase 07).

1. **Task 1: Fix Opus pricing and add MiniMax entry** - applied directly to ~/.claude/hooks/codex-pricing.js
2. **Task 2: Create and run migration script** - ~/.claude/hooks/migrate-opus-pricing.js created, executed, verified idempotent

**Plan metadata:** (final commit hash -- see below)

## Files Created/Modified

- `~/.claude/hooks/codex-pricing.js` - Fixed OPUS_PRICING ($15->$5, $75->$25); added minimax-m2.7 entry; updated header comment
- `~/.claude/hooks/migrate-opus-pricing.js` - New idempotent migration script; recalculates opus_baseline_usd from stored tokens using corrected rates; atomic write; dashboard regen; prints summary

## Decisions Made

- **Opus 4.6 pricing is $5/$25:** The $15/$75 rates were Opus 4.1 -- a 3x overstatement of the baseline. Dashboard savings percentages were wildly inflated (reporting ~86% when real savings are much lower).
- **MiniMax added to CODEX_PRICING (not a separate constant):** Consistent with existing pattern. `computeCodexCostStrict` and `computeCost` both work for minimax-m2.7 automatically without any function changes.
- **Migration rewrites global.jsonl only:** Per-project token-log.jsonl files never contain `opus_baseline_usd` -- the aggregator computes it at merge time from codex-pricing.js. No per-project file migration needed.
- **Idempotency by recalculation:** Script deterministically recalculates from stored token counts, not from the old stored `opus_baseline_usd` values. Running twice produces bitwise-identical output (confirmed via diff).

## Deviations from Plan

None - plan executed exactly as written.

## Issues Encountered

- Hook files (~/.claude/hooks/) are outside the project git repo, so per-task commits are not possible. This is an established pattern in this project (Phase 07 used the same approach). Planning artifact commits capture the metadata; actual hook changes are documented in the commit message.

## User Setup Required

None - no external service configuration required. Migration script ran fully automatically.

## Next Phase Readiness

- codex-pricing.js now has minimax-m2.7 pricing -- ready for minimax-exec.js (Plan 08-02) to `require('./codex-pricing')` and call `computeCodexCostStrict(tokens, 'minimax-m2.7')`
- global.jsonl has corrected opus_baseline_usd -- dashboard will now show realistic savings percentages
- dashboard.html regenerated and reflects corrected data

---
*Phase: 08-minimax-foundation*
*Completed: 2026-04-03*

## Self-Check: PASSED

- FOUND: ~/.claude/hooks/codex-pricing.js
- FOUND: ~/.claude/hooks/migrate-opus-pricing.js
- FOUND: ~/.claude/dashboard/dashboard.html
- FOUND: .planning/phases/08-minimax-foundation/08-01-SUMMARY.md
- Verified: OPUS_PRICING.input === 5 (true)
- Verified: CODEX_PRICING['minimax-m2.7'].input === 0.3 (true)
- Verified: computeCodexCostStrict returns 0.3 for minimax-m2.7 (not null)
- Verified: global.jsonl first record opus_baseline_usd === 0.086705 (match: true)
- Verified: global.jsonl count === 215 (>= 215 minimum)
- Verified: idempotency confirmed via diff (no differences on second run)
