---
phase: 02-model-executors-and-pricing
verified: 2026-04-04T00:00:00Z
status: human_needed
score: 12/12 must-haves verified (automated); 1 item needs human
re_verification: false
human_verification:
  - test: "Run websearch.sh with a real query and confirm JSON lines are returned"
    expected: "One JSON object per line with title, url, content fields"
    why_human: "SearXNG at 127.0.0.1:8888 is running but returned 0 results during automated check — all search engines unresponsive in current session. Script code is correct; external connectivity cannot be verified programmatically."
---

# Phase 2: Model Executors and Pricing — Verification Report

**Phase Goal:** All nine models are callable through a uniform interface; token costs are validated against the pricing table per provider with no shared formula
**Verified:** 2026-04-04
**Status:** human_needed — all automated checks passed; 1 behavioral item requires live search connectivity
**Re-verification:** No — initial verification

---

## Goal Achievement

### Observable Truths

| # | Truth | Status | Evidence |
|---|-------|--------|----------|
| 1 | pricing.js returns correct non-negative cost for all nine models | VERIFIED | All 9 PRICING keys present; computeAnthropicCost(1000,500,200,800,'claude-opus-4-6') = 0.02625; computeGeminiCost = 0.008; computeOpenAICost = 0.0009; computeFreeCost = 0; qwen and codex return 0 |
| 2 | token-logger.js normalizes four incompatible token schemas into canonical fields | VERIFIED | normalizeUsage tested for all four providers (anthropic, gemini, openai, ollama); correct field mapping confirmed |
| 3 | raw_usage is preserved in every token-logger record | VERIFIED | `raw_usage` field included verbatim in JSONL record at line 78 (COST-06 comment present) |
| 4 | codex-exec.js spawns Codex CLI with 300s timeout and returns unified shape | VERIFIED | 300000ms constant at line 111; execute/stream/available all exported; unified {success, output, tokens, cost, raw_usage, error} returned |
| 5 | minimax-exec.js calls MiniMax API via OpenAI SDK baseURL swap with temperature 0.01 and 90s timeout | VERIFIED | MINIMAX_BASE_URL = 'https://api.minimax.io/v1'; temperature = 0.01; timeoutMs = 90000; live call returned success: true with API key set |
| 6 | Both forked executors implement execute(), stream(), available() with no runWithFallback or fallback chain | VERIFIED | codex-exec exports {available, execute, stream}; minimax-exec exports {available, execute, stream}; runWithFallback is undefined on minimax-exec export; the only "runWithFallback" occurrence is a comment (line 17) |
| 7 | gemini-exec.js calls both Gemini models via @google/genai SDK with correct grounding and thinking mode | VERIFIED | googleSearch: {} used (not deprecated google_search_retrieval); ThinkingLevel.HIGH present; MODEL_MAP translates pricingKey to preview IDs; 90s TIMEOUT_MS; 429 retry backoff loop |
| 8 | qwen-exec.js uses inference probe in available() — not /api/tags — and returns false gracefully | VERIFIED | available() calls client.chat.completions.create with max_tokens:1; returned false in test (no GPU/ollama running); api/tags appears only in comments |
| 9 | perplexity-exec.js uses OpenAI SDK baseURL swap to https://api.perplexity.ai (no /v1 suffix) and supports MCP-first path | VERIFIED | PERPLEXITY_BASE_URL = 'https://api.perplexity.ai' (no /v1); MCP path returns mcpRequest object when opts.mcpAvailable === true; auth error returned correctly when key absent |
| 10 | No require() paths point outside the plugin directory (except the allowed openai global path) | VERIFIED | grep found zero ~/.claude/hooks/ references across all executors, lib, hooks, and tools directories |
| 11 | websearch.sh queries SearXNG at 127.0.0.1:8888 and is executable | VERIFIED | File exists, is executable (chmod +x), uses 127.0.0.1:8888 in SEARXNG_URL, no 0.0.0.0 in actual code (only in a comment warning) |
| 12 | fetchdocs.js calls Context7 via claude CLI and falls back to websearch.sh | VERIFIED | References 'context7' 6 times; falls back to websearch.sh via path.join(__dirname); file is executable |

**Automated Score:** 12/12 truths verified

---

## Required Artifacts

| Artifact | Expected | Status | Details |
|----------|----------|--------|---------|
| `~/.claude/plugins/seraphim/lib/pricing.js` | Per-provider cost functions for all nine models | VERIFIED | 224 lines; exports PRICING + 6 functions; 9 PRICING keys; per-provider functions (no shared formula) |
| `~/.claude/plugins/seraphim/hooks/token-logger.js` | Token logging hook with four-schema normalization | VERIFIED | 106 lines; imports normalizeUsage and computeCostForModel from ../lib/pricing; 10s timeout guard; SERAPHIM_RESULT marker; writes to .seraphim/token-log.jsonl |
| `~/.claude/plugins/seraphim/executors/codex-exec.js` | Codex CLI executor with unified interface | VERIFIED | 343 lines; exports {execute, stream, available}; 300s timeout; JSONL parsing; computeFreeCost for Codex |
| `~/.claude/plugins/seraphim/executors/minimax-exec.js` | MiniMax API executor with unified interface | VERIFIED | 264 lines; exports {execute, stream, available}; no runWithFallback; temperature 0.01; 90s AbortController; computeOpenAICost |
| `~/.claude/plugins/seraphim/executors/gemini-exec.js` | Gemini API executor with search grounding and thinking mode | VERIFIED | 279 lines; exports {execute, stream, available}; MODEL_MAP; googleSearch:{}; ThinkingLevel.HIGH; 429 backoff; computeGeminiCost |
| `~/.claude/plugins/seraphim/executors/qwen-exec.js` | Qwen local executor via ollama with inference probe and Forge mode | VERIFIED | 197 lines; exports {execute, stream, available}; inference probe in available(); FORGE_SYSTEM_PROMPT; parseForgeActions; 180s/60s timeouts; computeFreeCost |
| `~/.claude/plugins/seraphim/executors/perplexity-exec.js` | Perplexity Sonar executor with MCP+API dual path | VERIFIED | 203 lines; exports {execute, stream, available}; MCP-first via mcpRequest; no /v1 in baseURL; citation extraction; computeOpenAICost |
| `~/.claude/plugins/seraphim/package.json` | npm package manifest with @google/genai dependency | VERIFIED | "@google/genai": "^1.48.0" in dependencies; @google/generative-ai NOT present (not deprecated) |
| `~/.claude/plugins/seraphim/tools/websearch.sh` | SearXNG web search wrapper for non-Claude models | VERIFIED | Executable; queries 127.0.0.1:8888; returns JSONL with title/url/content |
| `~/.claude/plugins/seraphim/tools/fetchdocs.js` | Context7 documentation fetcher for non-Claude models | VERIFIED | Executable; 6 references to context7; falls back to websearch.sh |

---

## Key Link Verification

| From | To | Via | Status | Details |
|------|----|-----|--------|---------|
| token-logger.js | lib/pricing.js | require('../lib/pricing') | WIRED | Line 25: `require(path.join(__dirname, '../lib/pricing'))` |
| token-logger.js | .seraphim/token-log.jsonl | fs.appendFileSync | WIRED | Line 93: `fs.appendFileSync(logPath, ...)` with logPath pointing to .seraphim/token-log.jsonl |
| codex-exec.js | lib/pricing.js | require('../lib/pricing') | WIRED | Line 13: `require(path.join(__dirname, '../lib/pricing'))` |
| minimax-exec.js | lib/pricing.js | require('../lib/pricing') | WIRED | Line 21: `require(path.join(__dirname, '../lib/pricing'))` |
| gemini-exec.js | lib/pricing.js | require('../lib/pricing') | WIRED | Line 11: `require('../lib/pricing')` |
| qwen-exec.js | lib/pricing.js | require('../lib/pricing') | WIRED | Line 20: `require('../lib/pricing')` |
| perplexity-exec.js | lib/pricing.js | require('../lib/pricing') | WIRED | Line 26: `require('../lib/pricing')` |
| gemini-exec.js | @google/genai | require('@google/genai') | WIRED | Line 10: `require('@google/genai')`; node_modules/@google/genai present |
| websearch.sh | http://127.0.0.1:8888 | curl HTTP request | WIRED | Line 11: SEARXNG_URL="http://127.0.0.1:8888"; curl call uses this variable |
| perplexity-exec.js | https://api.perplexity.ai | OpenAI SDK baseURL swap | WIRED | Line 29: PERPLEXITY_BASE_URL = 'https://api.perplexity.ai' (no /v1) |
| qwen-exec.js | http://localhost:11434/v1 | OpenAI SDK baseURL swap | WIRED | Line 43: BASE_URL = 'http://localhost:11434/v1' |

---

## Data-Flow Trace (Level 4)

These are utility modules and helper scripts — not rendering components. Data flow is verified by confirming the compute path from API response to cost/token output.

| Artifact | Data Variable | Source | Produces Real Data | Status |
|----------|---------------|--------|--------------------|--------|
| pricing.js | computeCostForModel output | Per-provider formula applied to rawUsage | Yes — confirmed by spot-check (0.02625 for Opus, 0.008 for Gemini, 0 for free models) | FLOWING |
| token-logger.js | record.cost_usd | computeCostForModel(model, raw_usage) | Yes — uses pricing.js dispatcher | FLOWING |
| token-logger.js | record.tokens | normalizeUsage(raw_usage, provider) | Yes — four-schema normalization confirmed | FLOWING |
| minimax-exec.js | cost field | computeOpenAICost(prompt_tokens, completion_tokens, 'minimax-m2.7') | Yes — live API call confirmed success: true | FLOWING |

---

## Behavioral Spot-Checks

| Behavior | Command | Result | Status |
|----------|---------|--------|--------|
| pricing.js exports all 7 symbols | node -e "require('./lib/pricing')" | 7 exports confirmed | PASS |
| PRICING has exactly 9 model keys | Object.keys(PRICING).length | 9 | PASS |
| computeAnthropicCost returns positive | computeAnthropicCost(1000,500,200,800,'claude-opus-4-6') | 0.02625 | PASS |
| computeFreeCost returns 0 | computeFreeCost() | 0 | PASS |
| qwen and codex produce 0 via computeCostForModel | computeCostForModel('qwen-3.5-27b', {}) | 0 | PASS |
| normalizeUsage gemini schema | normalizeUsage({promptTokenCount:100, candidatesTokenCount:50}, 'gemini') | {input_tokens:100, output_tokens:50, thoughts_tokens:0} | PASS |
| normalizeUsage ollama schema | normalizeUsage({prompt_eval_count:100, eval_count:50}, 'ollama') | {input_tokens:100, output_tokens:50} | PASS |
| codex-exec exports {execute, stream, available} only | Object.keys(e).sort() | ['available','execute','stream'] | PASS |
| minimax-exec has no runWithFallback | typeof e.runWithFallback | undefined | PASS |
| minimax-exec has no isCodexRateLimited | typeof e.isCodexRateLimited | undefined | PASS |
| gemini-exec available() returns false without GEMINI_API_KEY | e.available() | false | PASS |
| gemini-exec returns auth error shape when no key | execute('test',{}) | {success:false, error_type:'auth', retriable:false, all fields present} | PASS |
| perplexity-exec returns auth error when no key | execute('test',{}) | {success:false, error_type:'auth'} | PASS |
| qwen available() returns false gracefully | e.available() | false (no GPU/ollama running, no throw) | PASS |
| no stale ~/.claude/hooks/ requires | grep across all executors/lib/hooks/tools | 0 matches | PASS |
| websearch.sh uses 127.0.0.1 not 0.0.0.0 | grep in SEARXNG_URL assignment | "http://127.0.0.1:8888" | PASS |
| @google/genai installed and importable | node -e "require('@google/genai')" | GoogleGenAI and ThinkingLevel loaded | PASS |
| websearch.sh returns search results | bash websearch.sh "nodejs" 2 | 0 results — SearXNG running but all engines unresponsive | ? HUMAN NEEDED |

---

## Requirements Coverage

| Requirement | Source Plan | Description | Status | Evidence |
|-------------|------------|-------------|--------|----------|
| EXEC-01 | 02-02 | codex-exec.js forked, unified interface, fallback chain removed | SATISFIED | Exports {execute, stream, available}; no runWithFallback; 300s timeout preserved |
| EXEC-02 | 02-02 | minimax-exec.js forked, unified interface, runWithFallback removed | SATISFIED | Exports {execute, stream, available}; runWithFallback undefined on exports; temperature 0.01; 90s timeout |
| EXEC-03 | 02-03 | gemini-exec.js with search grounding, thinking mode, 429 backoff | SATISFIED | googleSearch:{}; ThinkingLevel.HIGH; backoffDelay function; retry loop |
| EXEC-04 | 02-04 | qwen-exec.js with 180s timeout, warm-up probe, num_ctx 32768, JSON structured output | SATISFIED | EXECUTE_TIMEOUT_MS=180000; PROBE_TIMEOUT_MS=60000; NUM_CTX=32768; FORGE_SYSTEM_PROMPT; parseForgeActions |
| EXEC-05 | 02-04 | perplexity-exec.js via OpenAI SDK baseURL swap, citation extraction | SATISFIED | PERPLEXITY_BASE_URL='https://api.perplexity.ai'; response.citations extracted |
| EXEC-06 | 02-02, 02-03, 02-04 | Every executor implements execute, stream, available with {success, output, tokens, cost, error} shape | SATISFIED | All 5 executors export exactly {execute, stream, available}; return shape confirmed via spot-checks |
| EXEC-07 | 02-04 | qwen available() sends inference probe, returns false gracefully without GPU | SATISFIED | client.chat.completions.create with max_tokens:1; returned false without throw when ollama absent |
| EXEC-08 | 02-05 | websearch.sh wraps SearXNG at localhost:8888 | SATISFIED | Script exists, executable, uses 127.0.0.1:8888 |
| EXEC-09 | 02-05 | fetchdocs.js calls Context7 endpoint | SATISFIED | Script exists, executable, references context7 MCP, falls back to websearch.sh |
| COST-01 | 02-01 | lib/pricing.js exposes per-provider cost functions for all nine models — no shared formula | SATISFIED | Four separate functions (computeAnthropicCost, computeGeminiCost, computeOpenAICost, computeFreeCost); computeCostForModel dispatches by modelKey |
| COST-02 | 02-01 | token-logger.js handles four incompatible token schemas | SATISFIED | normalizeUsage covers anthropic/gemini/openai/ollama schemas; all four confirmed by spot-check |
| COST-06 | 02-01 | raw_usage preserved verbatim in token-logger records | SATISFIED | record.raw_usage = raw_usage (verbatim, line 78); COST-06 comment present |

All 12 required requirements are SATISFIED.

---

## Anti-Patterns Found

| File | Line | Pattern | Severity | Impact |
|------|------|---------|----------|--------|
| minimax-exec.js | 17 | "runWithFallback" in comment | INFO | Comment-only; not functional code; documents what was removed |
| qwen-exec.js | 7, 122, 125 | "api/tags" in comments | INFO | Comment-only; explicitly notes why /api/tags is NOT used |
| websearch.sh | 6 | "0.0.0.0" in comment text | INFO | Comment says "never 0.0.0.0"; actual SEARXNG_URL uses 127.0.0.1 |

No blockers or warnings. All anti-pattern hits are in comments that explain constraints — not functional issues.

---

## Human Verification Required

### 1. websearch.sh Returns Live Search Results

**Test:** Run `bash ~/.claude/plugins/seraphim/tools/websearch.sh "node.js http server" 5` in a terminal
**Expected:** Five or more lines of JSON, each with `title`, `url`, and `content` fields
**Why human:** SearXNG is running at 127.0.0.1:8888 and responds to HTTP (confirmed), but returned 0 results during automated check — all configured search engines appear unresponsive in the current network session. The script code is correct; this is an external dependency. When SearXNG has working search engines configured, the script will return results.

---

## Gaps Summary

No gaps. All twelve must-have truths are verified against the actual codebase. The one human-verification item (websearch.sh live results) is an external infrastructure dependency, not a code defect. The script is implemented correctly.

---

_Verified: 2026-04-04_
_Verifier: Claude (gsd-verifier)_
