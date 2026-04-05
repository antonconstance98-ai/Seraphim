---
phase: 02-model-executors-and-pricing
plan: 02
subsystem: executors
tags: [codex, minimax, executor, unified-interface, fork]
dependency_graph:
  requires: ["02-01"]
  provides: ["codex-exec.js unified executor", "minimax-exec.js unified executor"]
  affects: ["dispatch.js (will route to these)", "token-logger.js (consumes result shape)"]
tech_stack:
  added: []
  patterns:
    - "Unified executor interface: execute/stream/available exports only"
    - "Error shape: {error_type, message, retriable} — never throws"
    - "computeFreeCost for Codex (Plus subscription), computeOpenAICost for MiniMax"
    - "normalizeUsage('openai') for both — both use OpenAI-compat token schema"
key_files:
  created:
    - ~/.claude/plugins/seraphim/executors/codex-exec.js
    - ~/.claude/plugins/seraphim/executors/minimax-exec.js
  modified: []
decisions:
  - "Both executors delegate fallback logic to dispatch.js — no cross-executor dependencies"
  - "minimax-exec.js comment references removed functions by name for auditability — not functional code"
  - "Codex available() uses execSync('which codex') — synchronous check is safe since it exits immediately"
metrics:
  duration: "4 min"
  completed: "2026-04-05T03:32:39Z"
  tasks_completed: 2
  tasks_total: 2
  files_created: 2
  files_modified: 0
---

# Phase 02 Plan 02: Fork Codex and MiniMax Executors Summary

Forked `codex-exec.js` and `minimax-exec.js` from v2.0 hooks into the Seraphim plugin's unified executor interface. Both executors export exactly `{ execute, stream, available }` with the standardised `{success, output, tokens, cost, raw_usage, error}` result shape.

## Tasks Completed

| Task | Name | Commit | Files |
|------|------|--------|-------|
| 1 | Fork codex-exec.js to unified interface | 614381a | executors/codex-exec.js |
| 2 | Fork minimax-exec.js to unified interface | 826c510 | executors/minimax-exec.js |

## What Was Built

**codex-exec.js** — Codex GPT-5.4 CLI executor:
- Preserved: `runCodexExec()` (300s timeout, SIGTERM+SIGKILL), `parseCodexTokens()` (both JSONL formats)
- Added: `execute()`, `stream()`, `available()` unified wrappers
- Removed: `computeCost` re-export, `runGpt54MiniApi`, direct exposure of `runCodexExec`/`parseCodexTokens`
- Cost: always 0 via `computeFreeCost()` (ChatGPT Plus subscription)
- Error types: `unavailable` (ENOENT), `timeout` (300s), `rate_limit` (429/quota keywords), `unknown`

**minimax-exec.js** — MiniMax M-2.7 API executor:
- Preserved: `callWithRetry()` (exponential backoff+jitter), `runMinimax()` (OpenAI SDK baseURL swap, 90s timeout)
- Added: `execute()`, `stream()`, `available()` unified wrappers
- Removed: `runWithFallback()`, `isCodexRateLimited()`, `CODEX_EXEC_PATH` constant, `PRICING_PATH` old reference
- Temperature 0.01 preserved (MiniMax API rejects exactly 0)
- Cost: `computeOpenAICost(promptTokens, completionTokens, 'minimax-m2.7')`
- Error types: `auth` (no API key), `rate_limit` (429), `timeout` (AbortController), `unknown`

## Deviations from Plan

None — plan executed exactly as written.

The single acceptance-criteria concern (`grep -c "runWithFallback"` finds 1) is expected: the file header comment names both removed functions explicitly for auditability. The grep check in the plan verifies no functional code exports them, and `typeof e.runWithFallback === 'undefined'` confirms this.

## Known Stubs

None. Both executors are fully wired to `lib/pricing.js` for cost computation. No placeholder values or TODO stubs in the implementation path.

## Self-Check: PASSED

| Item | Status |
|------|--------|
| executors/codex-exec.js | FOUND |
| executors/minimax-exec.js | FOUND |
| 02-02-SUMMARY.md | FOUND |
| commit 614381a (codex-exec) | FOUND |
| commit 826c510 (minimax-exec) | FOUND |
