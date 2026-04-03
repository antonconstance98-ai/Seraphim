---
phase: 10-adversarial-plan-review
plan: 01
subsystem: hooks
tags: [minimax, codex, multi-round-review, reasoning-split, adversarial-review, model-routing]

# Dependency graph
requires:
  - phase: 08-minimax-foundation
    provides: minimax-exec.js with runMinimax(), codex-pricing.js with minimax-m2.7 pricing
  - phase: 09-dual-review-gate
    provides: codex-review-gate.js parallel review pattern, confirmed MiniMax API connectivity

provides:
  - minimax-exec.js v1.1.0 with reasoning_split opt-in and defensive response handling
  - codex-multi-round-reviewer.js v4.0.0 with MiniMax as Round 2 adversarial reviewer
  - D-08 fallback chain (MiniMax -> Codex) for Round 2 with partial-success guard
  - model field in review-state.json round records (backward-compatible)
  - per-model token logging with correct cost function per round

affects: [11-posttooluse-bug-scanner, 12-context-compression, 13-codex-execution-pipeline, 14-three-model-reporting]

# Tech tracking
tech-stack:
  added: []
  patterns:
    - "reasoning_split opt-in via opts.reasoningSplit — never hardcoded; backward-compatible"
    - "Defensive MiniMax response extraction: typeof guards + JSON.stringify fallbacks for null/array/object fields"
    - "D-08 partial-success guard: empty text treated same as failure (transport pass != feature pass)"
    - "Round 1 context capped at 4000 chars before injection — prevents prompt bloat and timeout"
    - "Model-per-round pattern: logTokens() and recordRoundResult() accept model param; default to gpt-5.4"
    - "Backward-compat resume: r1Record.model || 'gpt-5.4' — old state files resume without errors"

key-files:
  created: []
  modified:
    - "~/.claude/hooks/minimax-exec.js (v1.0.0 -> v1.1.0) — reasoning_split support with defensive extraction"
    - "~/.claude/hooks/codex-multi-round-reviewer.js (v3.0.0 -> v4.0.0) — MiniMax Round 2 routing"

key-decisions:
  - "reasoning_split added as opt-in via opts.reasoningSplit (not hardcoded) — callers that don't pass it get unchanged behavior"
  - "Defensive response extraction guards null, array, and object shapes in message.content and reasoning fields — MiniMax API field naming is inconsistent across docs"
  - "Partial success guard: r2Result.text.trim().length === 0 triggers D-08 fallback — empty text from MiniMax is a feature-level failure even when success:true"
  - "Round 1 context capped at MAX_R1_CONTEXT=4000 chars — prevents MiniMax timeout/truncation on long constructive reviews"
  - "computeCodexCostStrict (not computeCost) for MiniMax token logging — avoids gpt-5.4 rate misapplication (confirmed minimax-m2.7 pricing exists from Phase 8)"
  - "No runWithFallback() for Round 2 — Phase 9 pattern goes Codex->MiniMax; Phase 10 goes MiniMax->Codex (opposite direction)"

patterns-established:
  - "Pattern: Model-per-round routing — runMultiRoundReview() hides model selection internally; callers unchanged"
  - "Pattern: Partial-success guard — check both .success AND .text.trim().length before accepting a model result"
  - "Pattern: Round 1 context injection — cap at 4000 chars, pass as preamble with explicit 'not bound by' framing"

requirements-completed: [D-01, D-03, D-04, D-05, D-06, D-07, D-08]

# Metrics
duration: 5min
completed: 2026-04-03
---

# Phase 10 Plan 01: Adversarial Plan Review Summary

**MiniMax M-2.7 wired as Round 2 adversarial reviewer with reasoning_split support, D-08 Codex fallback, and per-model token logging**

## Performance

- **Duration:** 5 min
- **Started:** 2026-04-03T19:31:05Z
- **Completed:** 2026-04-03T19:36:23Z
- **Tasks:** 2
- **Files modified:** 2 (outside repo: ~/.claude/hooks/)

## Accomplishments

- minimax-exec.js v1.1.0: reasoning_split opt-in with defensive response extraction (null/array/object guards, both reasoning field names, JSON.stringify fallback, think-tag wrapping)
- codex-multi-round-reviewer.js v4.0.0: Round 2 now routes to MiniMax M-2.7 via runMinimax() with reasoningSplit:true and maxTokens:4000; Round 1 remains Codex unchanged
- D-08 fallback triggers on MiniMax failure OR empty text (partial success guard) with Codex normalization to common result shape
- D-05 prompt context: Round 1 findings capped at 4000 chars and passed as preamble to Round 2 with explicit "not bound by" framing
- D-06/D-07: model field in round records and logTokens uses correct cost function per model (computeCodexCostStrict for minimax-m2.7, computeCost for gpt-5.4)
- Backward-compatible: old review-state.json files without model field resume without errors

## Task Commits

Each task was committed atomically:

1. **Task 1: Add reasoning_split support to minimax-exec.js** - `97a30ef` (feat)
2. **Task 2: Route Round 2 to MiniMax in codex-multi-round-reviewer.js** - `00f25a3` (feat)

_Note: Hook files live at ~/.claude/hooks/ (outside repo). Commits use --allow-empty with descriptive body annotations following the Phase 9 convention established in commit a30d4fa._

## Files Created/Modified

- `~/.claude/hooks/minimax-exec.js` (v1.0.0 -> v1.1.0) — reasoning_split opt-in, defensive content/reasoning extraction, think-tag wrapping, backward-compatible
- `~/.claude/hooks/codex-multi-round-reviewer.js` (v3.0.0 -> v4.0.0) — MiniMax Round 2 routing, D-08 fallback, MAX_R1_CONTEXT cap, model-per-round tracking, updated logTokens and recordRoundResult

## Decisions Made

- `reasoning_split` is opt-in via `opts.reasoningSplit` (never hardcoded) — callers that don't pass it (codex-review-gate.js, minimax-post-scan.js) are entirely unaffected
- Defensive extraction guards both `reasoning_content` and `reasoning_details` field names — MiniMax API field naming was inconsistent across synthesis docs; defensive pattern covers both
- Partial success guard treats empty MiniMax text as a feature-level failure and triggers D-08 fallback — "success:true but empty text" is a transport-level pass that should not silently degrade review quality
- `computeCodexCostStrict` confirmed to already have `minimax-m2.7` pricing from Phase 8 ($0.30/$1.20/M) — no pricing changes needed
- Adversarial prompt updated to red-team framing with 6 explicit failure-mode questions, more aggressive than the previous "skeptical reviewer" wording

## Deviations from Plan

None — plan executed exactly as written. All 7 modifications to codex-multi-round-reviewer.js and all minimax-exec.js changes matched the plan spec precisely. Both automated verification suites (14 checks for reviewer, 9 checks for minimax-exec) passed on first attempt.

## Issues Encountered

None. The hook files at `~/.claude/hooks/` are outside the git repo — this was already established as a known constraint in Phase 9, and the commit annotation convention (describing external file changes in the commit body) was already in place.

## Known Stubs

None — all wiring is complete. The MiniMax Round 2 path calls runMinimax() with real opts; the D-08 fallback calls runCodexExec() with real opts. No placeholder data flows to any output.

## User Setup Required

None — no new environment variables, services, or configuration required. MINIMAX_API_KEY was set in Phase 8.

## Next Phase Readiness

- Phase 10 Plan 02 (if exists): codex-plan-reviewer.js and codex-superpowers-plan-reviewer.js callers may need REVIEWS.md header updates to reflect "Round 1: gpt-5.4, Round 2: minimax-m2.7" (identified as Open Question #3 in research, Claude's discretion)
- Phase 11 (PostToolUse bug scanner): minimax-exec.js v1.1.0 is ready; reasoning_split is available as opt-in for scan results if transparency is desired
- All downstream phases can safely require() codex-multi-round-reviewer.js — runMultiRoundReview() signature unchanged

---
*Phase: 10-adversarial-plan-review*
*Completed: 2026-04-03*
