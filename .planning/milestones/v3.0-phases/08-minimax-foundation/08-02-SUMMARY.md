---
phase: 08-minimax-foundation
plan: 02
subsystem: infra
tags: [minimax, openai-sdk, baseurl-swap, fallback-chain, rate-limit-detection, hooks]

# Dependency graph
requires:
  - phase: 08-01
    provides: minimax-m2.7 pricing in codex-pricing.js (computeCodexCostStrict)

provides:
  - runMinimax() — MiniMax M-2.7 API call via OpenAI SDK with baseURL swap
  - runWithFallback() — Codex CLI -> MiniMax API -> fail-open/fail-closed chain
  - isCodexRateLimited() — 4-method rate-limit detection from runCodexExec result

affects:
  - 09-dual-review-gate
  - 10-adversarial-plan-review
  - 11-posttooluse-bug-scanner
  - 12-context-compression
  - 13-codex-execution-pipeline

# Tech tracking
tech-stack:
  added: []
  patterns:
    - "Lazy-require OpenAI SDK: try standard path, fallback to global npm path (mirrors codex-exec.js)"
    - "AbortController with clearTimeout(timer) in finally — same timeout pattern as runGpt54MiniApi"
    - "callWithRetry: exponential backoff (1s*2^attempt, cap 8s) + random jitter (0-500ms), 3 retries"
    - "Fail-open vs fail-closed split by taskCategory — review tasks never block workflow, execution tasks require user intervention"

key-files:
  created:
    - ~/.claude/hooks/minimax-exec.js
  modified: []

key-decisions:
  - "callWithRetry wraps only the API call, not the full runMinimax function — timeout abort still clears in finally regardless of retry count"
  - "isCodexRateLimited: 4 detection methods cover all D-15 conditions including timeout-with-no-output (weekly cap signal)"
  - "runWithFallback returns immediately on non-rate-limit Codex failure — only escalates to MiniMax on rate-limit (D-16 scope boundary)"
  - "Defensive fallback for cached_tokens: checks prompt_tokens_details.cached_tokens then usage.cached_tokens — handles MiniMax API field name uncertainty (Open Question #1)"

patterns-established:
  - "MiniMax module pattern: same library module shape as codex-exec.js (no shebang, no stdin, no stdout, 'use strict')"
  - "Source attribution: 'codex-cli' | 'api-fallback' | 'codex-failed' | 'all-failed-review' | 'all-failed-execution'"

requirements-completed: [D-01, D-02, D-03, D-04, D-05, D-06, D-14, D-15, D-16, D-17]

# Metrics
duration: 4min
completed: 2026-04-03
---

# Phase 08 Plan 02: MiniMax Foundation Summary

**MiniMax M-2.7 provider module with OpenAI SDK baseURL swap, exponential-backoff retry, and a three-tier fallback chain (Codex CLI -> MiniMax API -> fail-open/closed by taskCategory)**

## Performance

- **Duration:** ~4 min
- **Started:** 2026-04-03T18:12:28Z
- **Completed:** 2026-04-03T18:15:44Z
- **Tasks:** 1
- **Files modified:** 1

## Accomplishments

- Created `~/.claude/hooks/minimax-exec.js` — 287-line shared library module (not a hook script)
- `runMinimax()` calls MiniMax M-2.7 via OpenAI SDK with `baseURL: https://api.minimax.io/v1`, temperature 0.01, max_tokens 2000, 90s timeout, 3-retry exponential backoff with jitter
- `isCodexRateLimited()` detects all 4 D-15 conditions: rateLimitPct >= 95, stderr keywords (rate limit/quota/usage limit), HTTP 429 in JSONL, timeout with no output
- `runWithFallback()` implements the full Codex CLI -> MiniMax API -> fail-open/fail-closed chain with source attribution on every return path

## Task Commits

Hook file is outside the git repository (`~/.claude/hooks/`) — same pattern as all previous hook scripts. No per-task commit; documented in plan metadata commit.

1. **Task 1: Create minimax-exec.js** — installed at `~/.claude/hooks/minimax-exec.js` (no git hash; outside repo)

**Plan metadata:** _(committed in final docs commit below)_

## Files Created/Modified

- `~/.claude/hooks/minimax-exec.js` — MiniMax provider module: runMinimax, runWithFallback, isCodexRateLimited (287 lines)

## Decisions Made

- `callWithRetry` wraps only the API call, not the full function — ensures the AbortController timer always fires and clears in `finally` even if retry loops run long
- `runWithFallback` escalates to MiniMax only on rate-limit failure (not all Codex failures) — prevents MiniMax API spend for misconfigurations or auth errors
- Defensive double-check for `cached_tokens` field: `prompt_tokens_details.cached_tokens || usage.cached_tokens` — MiniMax API field placement is undocumented in their compat layer

## Deviations from Plan

None — plan executed exactly as written.

## Issues Encountered

None. Module loaded cleanly; all 6 `isCodexRateLimited` test cases passed; `runMinimax` API key guard returned correct shape without MINIMAX_API_KEY set.

One verification check reported a false failure (MINIMAX_BASE_URL string comparison with alignment spaces). The URL `'https://api.minimax.io/v1'` is correct in the file — the test was doing an exact string match against `MINIMAX_BASE_URL =` (single space) while the file uses `MINIMAX_BASE_URL   =` (aligned spacing). Verified separately that the URL value and `baseURL:` reference are both correct.

## User Setup Required

None — MINIMAX_API_KEY will be configured in Plan 03 (connectivity verification step).

## Next Phase Readiness

- `minimax-exec.js` is ready for Phases 9-14 to `require()` it
- All three exports verified: runMinimax (function), runWithFallback (function), isCodexRateLimited (function)
- No new npm dependencies — uses existing openai SDK at global path
- MINIMAX_API_KEY not yet set on this machine — Plan 03 handles environment configuration and live connectivity test

---
*Phase: 08-minimax-foundation*
*Completed: 2026-04-03*
