---
phase: 05-data-pipeline
plan: 01
subsystem: infra
tags: [nodejs, pricing, hooks, refactor, codex-exec, cost-reporting]

# Dependency graph
requires: []
provides:
  - codex-pricing.js shared module with CODEX_PRICING, OPUS_PRICING, computeCost, computeCodexCostStrict, computeOpusCost
  - Centralized single source of truth for all model pricing constants
  - computeCodexCostStrict: strict mode returning null for unknown models (for Phase 5 Plan 02 aggregator)
  - codex-exec.js re-exports computeCost via codex-pricing.js (codex-token-logger.js import chain intact)
  - codex-cost-reporter.js imports computeOpusCost from codex-pricing.js (no inline constants)
affects:
  - 05-02 (codex-global-aggregator.js will import computeCodexCostStrict from codex-pricing.js)
  - 06-dashboard (dashboard consumes pricing via aggregator)

# Tech tracking
tech-stack:
  added: []
  patterns:
    - "Shared pricing module pattern: pricing constants live in one place, consumers require() and destructure"
    - "Backward-compatible re-export: codex-exec.js re-exports computeCost so downstream import chains need no changes"
    - "Two-tier cost functions: computeCost (fallback, always number) vs computeCodexCostStrict (null for unknowns)"

key-files:
  created:
    - ~/.claude/hooks/codex-pricing.js
  modified:
    - ~/.claude/hooks/codex-exec.js
    - ~/.claude/hooks/codex-cost-reporter.js

key-decisions:
  - "Kept CODEX_PRICING destructured in codex-exec.js require even though unused there — explicit re-export pattern documents the module relationship"
  - "computeOpusCost preserves no-rounding behavior from original codex-cost-reporter.js — stored cost values unchanged"
  - "computeCodexCostStrict returns null for unknown models — new consumers surface pricing gaps instead of silently using wrong pricing"

patterns-established:
  - "Pricing module pattern: single codex-pricing.js owns all constants; consumers require() and destructure only what they need"
  - "Re-export pattern: codex-exec.js passes computeCost through module.exports so codex-token-logger.js import chain needs zero changes"

requirements-completed: [PIPE-02]

# Metrics
duration: 2min
completed: 2026-04-03
---

# Phase 5 Plan 01: Centralized Pricing Module Summary

**Single codex-pricing.js module centralizes all GPT and Opus pricing constants, with backward-compatible re-export through codex-exec.js preserving the codex-token-logger.js import chain**

## Performance

- **Duration:** 2 min
- **Started:** 2026-04-03T00:11:34Z
- **Completed:** 2026-04-03T00:14:22Z
- **Tasks:** 2
- **Files modified:** 3 (system files at ~/.claude/hooks/)

## Accomplishments

- Created `~/.claude/hooks/codex-pricing.js` with 5 exports: CODEX_PRICING, OPUS_PRICING, computeCost, computeCodexCostStrict, computeOpusCost
- Refactored `codex-exec.js` to import from shared module and re-export computeCost unchanged (codex-token-logger.js chain unbroken)
- Refactored `codex-cost-reporter.js` to import computeOpusCost and OPUS_PRICING from shared module (no inline constants remain)
- 35 behavioral checks passed across both tasks; 6 plan-level verification checks all pass

## Task Commits

Each task was committed atomically:

1. **Tasks 1+2: Create codex-pricing.js and refactor both hooks** - `4a06bfa` (feat)

Note: Tasks 1 and 2 were committed together because both modify system files at `~/.claude/hooks/` which are outside the project git repo. The commit documents all three system file changes with full detail.

**Plan metadata:** (docs commit follows)

## Files Created/Modified

- `~/.claude/hooks/codex-pricing.js` - New centralized pricing module; exports CODEX_PRICING, OPUS_PRICING, computeCost, computeCodexCostStrict, computeOpusCost
- `~/.claude/hooks/codex-exec.js` - Removed inline PRICING constant and computeCost function; added require('./codex-pricing'); module.exports unchanged (re-exports computeCost)
- `~/.claude/hooks/codex-cost-reporter.js` - Removed inline OPUS_PRICING constant and computeOpusCost function; added require('./codex-pricing')

## Decisions Made

- computeOpusCost preserves the no-rounding behavior from the original codex-cost-reporter.js implementation — this avoids changing any stored cost values in existing token-log.jsonl files
- computeCodexCostStrict was added alongside computeCost (not as a replacement) to serve new consumers (Phase 5 Plan 02 aggregator) that need to surface pricing gaps explicitly, while keeping all existing consumers working without change
- CODEX_PRICING is destructured in codex-exec.js require even though it is not used in the file body — this makes the module relationship explicit and matches the plan spec exactly

## Deviations from Plan

None — plan executed exactly as written.

The only noteworthy implementation detail: Tasks 1 and 2 were committed in a single git commit because all three hook files are system files at `~/.claude/hooks/` (outside the project repository). This is consistent with how prior phases handled system file changes (see commit `5829696` for precedent). Not a deviation — no behavior or output differs from plan spec.

## Issues Encountered

None.

## Known Stubs

None — this plan is a pure refactor. All pricing constants and functions produce identical outputs to the originals. No stub values, placeholders, or unwired data paths.

## User Setup Required

None — no external service configuration required. The hook files are already in place at `~/.claude/hooks/` and will be loaded automatically by all hook scripts on next invocation.

## Next Phase Readiness

- codex-pricing.js is ready for Phase 5 Plan 02 (codex-global-aggregator.js) to `require('./codex-pricing')` and use computeCodexCostStrict for per-record cost computation
- codex-token-logger.js import chain verified intact — existing token logging continues to work unchanged
- codex-cost-reporter.js session reports continue to produce identical output — no stored data affected

---
*Phase: 05-data-pipeline*
*Completed: 2026-04-03*
