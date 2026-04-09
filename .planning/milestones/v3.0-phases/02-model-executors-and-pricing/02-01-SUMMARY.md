---
phase: 02-model-executors-and-pricing
plan: "01"
subsystem: pricing-and-token-logging
tags: [pricing, token-logging, cost-tracking, four-schema-normalization]
dependency_graph:
  requires: [01-plugin-scaffold-and-infrastructure]
  provides: [lib/pricing.js, hooks/token-logger.js]
  affects: [all-executors, token-log.jsonl]
tech_stack:
  added: []
  patterns: [per-provider-cost-functions, four-schema-normalization, stdin-hook-pattern, jsonl-append-log]
key_files:
  created:
    - ~/.claude/plugins/seraphim/lib/pricing.js
    - ~/.claude/plugins/seraphim/hooks/token-logger.js
  modified: []
decisions:
  - "cache_read tokens are a positive charge at reduced rate — not a credit or discount (COST-01)"
  - "normalizeUsage dispatcher routes on provider string, not model key — provider is the stable identifier across API response shapes"
  - "computeCostForModel routes on modelKey to pick correct provider function — nine models, four provider paths"
  - "token-logger.js writes to .seraphim/token-log.jsonl (not .planning/) per Seraphim per-project state convention"
  - "stdinTimeout.unref() on the 10s guard ensures the timer does not block normal process exit"
metrics:
  duration_seconds: 115
  completed_date: "2026-04-05T03:28:03Z"
  tasks_completed: 2
  tasks_total: 2
  files_created: 2
  files_modified: 0
---

# Phase 02 Plan 01: Pricing Module and Token Logger Summary

**One-liner:** Per-provider cost functions for all nine models with four-schema token normalization and JSONL logging.

## What Was Built

### lib/pricing.js

Central pricing module covering all nine Seraphim models. Exports:

- `PRICING` — constant with exact per-1M-token rates for all nine `pricingKey` values from `models.json`
- `computeAnthropicCost(inputTokens, cacheReadTokens, cacheWriteTokens, outputTokens, modelKey)` — Anthropic-specific: cache_read is a positive charge at reduced rate, cache_write at elevated rate
- `computeGeminiCost(promptTokenCount, candidatesTokenCount, modelKey)` — Gemini-specific: simple input+output formula
- `computeOpenAICost(promptTokens, completionTokens, modelKey)` — OpenAI-compatible: MiniMax, Perplexity, Codex API
- `computeFreeCost()` — returns 0 for Codex (Plus subscription) and Qwen (local GPU)
- `computeCostForModel(modelKey, rawUsage)` — dispatcher routing all nine models to the correct provider function
- `normalizeUsage(rawUsage, provider)` — maps four incompatible provider token schemas to canonical `{input_tokens, output_tokens, ...}` fields

All cost functions return `parseFloat(cost.toFixed(6))` — six decimal places.

### hooks/token-logger.js

PostToolUse hook for Seraphim. Reads `[SERAPHIM_RESULT]` markers from tool results, normalizes token usage across all four provider schemas, computes cost via `pricing.js`, and appends a JSONL record to `.seraphim/token-log.jsonl`.

Key properties:
- 10-second stdin timeout guard with `.unref()` (does not block normal process exit)
- Silent fail on all error paths (hooks must never block)
- `raw_usage` preserved verbatim in every record (COST-06 compliance)
- Writes to `.seraphim/` not `.planning/` — Seraphim per-project state convention
- No dependency on `~/.claude/hooks/` — standalone plugin constraint satisfied

## Verification Results

All acceptance criteria passed:

- `PRICING` contains exactly 9 keys matching `models.json` pricingKey values
- `computeAnthropicCost(1000, 500, 200, 800, 'claude-opus-4-6')` returns `0.02625` (positive, non-zero)
- `computeGeminiCost(1000, 500, 'gemini-3.1-pro')` returns `0.008` (positive, non-zero)
- `computeOpenAICost(1000, 500, 'minimax-m2.7')` returns `0.0009` (positive, non-zero)
- `computeFreeCost()` returns exactly `0`
- `normalizeUsage({promptTokenCount: 100, candidatesTokenCount: 50}, 'gemini')` returns `{input_tokens: 100, output_tokens: 50, thoughts_tokens: 0}`
- `normalizeUsage({prompt_eval_count: 100, eval_count: 50}, 'ollama')` returns `{input_tokens: 100, output_tokens: 50}`
- `computeCostForModel('qwen-3.5-27b', {})` returns `0`
- No `require()` calls to paths outside the plugin directory
- `token-logger.js` contains `setTimeout` timeout guard
- `token-logger.js` detects `[SERAPHIM_RESULT]` marker (not `[CODEX_RESULT]`)
- `token-logger.js` imports from `../lib/pricing` (relative path within plugin)
- `token-logger.js` writes to `.seraphim/token-log.jsonl`

## Commits

| Task | Commit | Message |
|------|--------|---------|
| Task 1: pricing.js | 3ead6ef | feat(02-01): create lib/pricing.js with per-provider cost functions for all nine models |
| Task 2: token-logger.js | 0d7fafa | feat(02-01): create hooks/token-logger.js with four-schema normalization and raw_usage preservation |

## Deviations from Plan

None — plan executed exactly as written.

## Known Stubs

None — both files are fully wired. pricing.js uses exact rates from the plan specification. token-logger.js routes to real normalizeUsage and computeCostForModel functions.

## Self-Check: PASSED

- `/home/alucard/.claude/plugins/seraphim/lib/pricing.js` exists
- `/home/alucard/.claude/plugins/seraphim/hooks/token-logger.js` exists
- Commit `3ead6ef` exists in plugin repo
- Commit `0d7fafa` exists in plugin repo
