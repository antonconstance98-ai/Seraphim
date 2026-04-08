# Phase 8: MiniMax Foundation - Context

**Gathered:** 2026-04-03
**Status:** Ready for planning

<domain>
## Phase Boundary

Add MiniMax M-2.7 as a model provider in the hook infrastructure. Create the shared SDK wrapper module, add MiniMax to the centralized pricing module, fix the Opus pricing error, configure environment/settings, build the Codex→MiniMax fallback chain, and verify connectivity. This is the foundation all subsequent v2.0 phases depend on.

</domain>

<decisions>
## Implementation Decisions

### Module architecture
- **D-01:** Standalone `minimax-exec.js` module, separate from `codex-exec.js`. Clean separation of concerns — each model provider has its own module.
- **D-02:** Exports `runMinimax(prompt, opts)` as the primary function. Uses the existing `openai` npm package (v6.33.0, already installed) with `baseURL: https://api.minimax.io/v1`.
- **D-03:** Temperature set to `0.01` (MiniMax API rejects exactly `0`; range must be `(0.0, 1.0]`).
- **D-04:** Default `max_tokens: 2000` to control MiniMax's verbosity tax (~4x more output tokens than median models).
- **D-05:** Returns `{ success, text, tokens, cost }` shape — `cost` computed via `codex-pricing.js`.
- **D-06:** Also exports `runWithFallback(prompt, opts)` — tries Codex CLI first, falls back to MiniMax on rate-limit errors, then prompts user as last resort. Built into the foundation module from day one so Phases 9-12 can benefit from fallback resilience immediately.

### Pricing correction
- **D-07:** Fix `OPUS_PRICING` in `codex-pricing.js` from `{ input: 15.00, output: 75.00 }` (Opus 4.1) to `{ input: 5.00, output: 25.00 }` (Opus 4.6). Cached input from `3.75` to `1.25`.
- **D-08:** Add MiniMax pricing: `'minimax-m2.7': { input: 0.30, cached_input: 0.06, output: 1.20 }`.
- **D-09:** Full historical recalculation — write a migration script that re-processes all `token-log.jsonl` files across all projects, recalculates Opus baseline costs with corrected pricing, and regenerates `global.jsonl`. Dashboard savings percentages will change.

### Config & API key
- **D-10:** Separate `minimax` block in project `.claude/settings.json`, alongside the existing `codex` block. Not nested inside `codex`.
- **D-11:** Config fields: `enabled`, `model`, `api_key_env`, `base_url`, `max_tokens_default`, `tasks` (array of task types routed to MiniMax).
- **D-12:** Use Pay-As-You-Go API key (not Token Plan). Key never expires. $25 in credits already available. Evaluate Token Plan subscription after Phases 9-12 are running if usage warrants it.
- **D-13:** API key stored as `MINIMAX_API_KEY` environment variable. Never in plaintext files.

### Fallback wiring
- **D-14:** Fallback chain built into `minimax-exec.js` as `runWithFallback(prompt, opts)`.
- **D-15:** Codex rate-limit detection via: exit code + stderr "rate limit"/"quota"/"usage limit", HTTP 429 in JSONL output, timeout with no output, `rate_limit_pct >= 95`.
- **D-16:** On Codex failure: log the failure reason to `token-log.jsonl`, auto-retry via MiniMax API, log MiniMax usage as `source: 'api-fallback'`. On MiniMax failure: prompt user with both error messages and options (wait/retry, check key, skip task).
- **D-17:** Execution fallback is fail-closed (prompt user). Review fallback is fail-open (skip if both fail).

### Claude's Discretion
- Error handling and retry logic details (exponential backoff, jitter, max retries)
- Migration script implementation approach (batch vs streaming)
- Exact format of fallback log entries in token-log.jsonl
- Whether to add a `minimax-m2.7-highspeed` pricing entry alongside standard

</decisions>

<specifics>
## Specific Ideas

- The `runMinimax()` function should mirror the preview code shown during discussion — simple, direct, using the existing `openai` package with a `baseURL` swap
- MiniMax's `<think>` tags and `reasoning_split` are important for later phases (adversarial review) but not needed in Phase 8 foundation — just basic chat completion
- The migration script for pricing correction should be idempotent (safe to run multiple times)
- Dashboard will show different savings numbers after the pricing fix — this is expected and correct

</specifics>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### MiniMax API & Integration
- `minimax-m2.7-synthesis.md` — Complete research synthesis: API endpoints, pricing, benchmarks, gotchas, integration patterns
- `minimax-m2.7-research.md` — Raw Claude research with detailed API specs, rate limits, platform availability
- `research/deep-research-report.md` — Node.js integration deep-dive: SDK patterns, streaming, tool calling, retry policy
- `research/deep-research-report(1).md` — Developer ecosystem: IDE support, MCP servers, community sentiment

### Existing Hook Infrastructure
- `~/.claude/hooks/codex-exec.js` — Execution module pattern to follow (exports, error handling, token parsing)
- `~/.claude/hooks/codex-pricing.js` — Pricing module to extend (CODEX_PRICING dict, computeCost functions)
- `~/.claude/hooks/codex-review-gate.js` — Stop hook that will consume minimax-exec.js in Phase 9
- `~/.claude/hooks/codex-multi-round-reviewer.js` — Plan reviewer that will consume minimax-exec.js in Phase 10

### Project Config
- `.claude/settings.json` — Project-level codex config block to add minimax block alongside
- `~/.claude/settings.json` — Global hooks registration (no changes needed in Phase 8)

### Cost Data
- `~/.claude/dashboard/global.jsonl` — Global aggregated token data (needs recalculation)
- `.planning/token-log.jsonl` — Per-project token log (needs recalculation)

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `codex-pricing.js` — Already has the dict pattern (`CODEX_PRICING`) and two cost functions (`computeCost` for backward-compat, `computeCodexCostStrict` for strict). Add MiniMax entry to `CODEX_PRICING`.
- `codex-exec.js` — Has `runGpt54MiniApi()` which is a direct OpenAI SDK API call — closest pattern to what `runMinimax()` needs.
- `openai` npm package v6.33.0 — Already installed. MiniMax uses same SDK with `baseURL` swap.

### Established Patterns
- All hooks use `'use strict'` and stdin-based JSON input with timeout guards
- Shared modules export functions, not classes
- Token logging uses JSONL format with consistent schema: `{ timestamp, session_id, model, source, task_type, tokens: { input, cached_input, output }, cost_usd }`
- Cost functions use `parseFloat(cost.toFixed(6))` for 6 decimal place precision
- `computeCost` always returns a number (never null) — backward compatible fallback
- `computeCodexCostStrict` returns null for unknown models — used for gap detection

### Integration Points
- `codex-pricing.js` is required by: `codex-exec.js`, `codex-cost-reporter.js`, `codex-global-aggregator.js`, `codex-token-logger.js`
- New `minimax-exec.js` will be required by: `codex-review-gate.js` (Phase 9), `codex-multi-round-reviewer.js` (Phase 10), `minimax-post-scan.js` (Phase 11), `minimax-compress.js` (Phase 12), gsd-executor (Phase 13)
- `codex-global-aggregator.js` discovers token-log.jsonl via configurable root paths — no changes needed for discovery, only for pricing computation

</code_context>

<deferred>
## Deferred Ideas

- Token Plan evaluation — revisit after Phases 9-12 are running to see if usage justifies the $10/mo subscription vs pay-as-you-go
- MiniMax `reasoning_split: true` support — needed for Phase 10 (adversarial review) but not Phase 8
- `minimax-m2.7-highspeed` variant ($0.60/$2.40) — add pricing entry if latency-sensitive use cases emerge
- OpenRouter as a secondary MiniMax endpoint (abstraction layer for model swapping) — evaluate if MiniMax direct API has reliability issues

</deferred>

---

*Phase: 08-minimax-foundation*
*Context gathered: 2026-04-03*
