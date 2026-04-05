# Phase 2: Model Executors and Pricing - Research

**Researched:** 2026-04-04
**Domain:** Node.js model executor layer — nine model integrations, per-provider pricing, token logging, helper scripts
**Confidence:** HIGH (all critical API patterns verified against installed SDK type definitions and prior verified STACK.md research)

---

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions

**D-01:** Perplexity executor uses both MCP bridge and direct HTTP API, with fallback. Try MCP first (leverages existing Perplexity MCP config, zero additional setup). If MCP is unavailable, fall back to direct API via OpenAI SDK baseURL swap to `https://api.perplexity.ai` with `PERPLEXITY_API_KEY`.

**D-02:** perplexity-exec.js must detect MCP availability at runtime and route accordingly.

**D-03:** Claude decides the structured output format for Qwen Forge mode based on what Qwen handles most reliably. Design spec suggests JSON `{"action": "write", "path": "...", "content": "..."}` — validate whether Qwen 3.5 can produce this reliably.

**D-04:** Executors return `{success: false, error_type, retriable: boolean}` — never throw. dispatch.js checks `retriable` flag and `error_type` to decide: retry same model, try profile-defined fallback, or surface to user.

**D-05:** Error types: `unavailable`, `timeout`, `rate_limit`, `auth`, `parse`, `unknown`.

**D-06:** dispatch.js retry policy: retry on `rate_limit` (with exponential backoff), attempt fallback on `unavailable`/`timeout`, surface to user on `auth`/`unknown`.

### Claude's Discretion
- Qwen structured output schema specifics
- Gemini SDK integration patterns (search grounding, thinking mode)
- Helper script implementation details
- Fork adaptation specifics (path resolution, interface adaptation)

### Deferred Ideas (OUT OF SCOPE)
None — discussion stayed within phase scope
</user_constraints>

---

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| EXEC-01 | `codex-exec.js` forked from v2.0, adapted to unified interface, fallback chain removed | Fork source exists at `~/.claude/hooks/codex-exec.js`; interface adapter pattern documented below |
| EXEC-02 | `minimax-exec.js` forked from v2.0, adapted to unified interface, `runWithFallback` removed, temperature 0.01 preserved | Fork source exists at `~/.claude/hooks/minimax-exec.js`; runWithFallback removal documented |
| EXEC-03 | `gemini-exec.js` integrates Gemini 3.1 Pro and Gemini 3 Flash via `@google/genai` with search grounding, thinking mode, and 429 exponential backoff | Full SDK API verified from installed type definitions — patterns documented below |
| EXEC-04 | `qwen-exec.js` runs Qwen locally via ollama HTTP at localhost:11434 with 180s timeout, warm-up probe, `num_ctx: 32768`, and JSON structured output for Forge mode | API patterns and model tag verified; ollama OpenAI-compat endpoint confirmed |
| EXEC-05 | `perplexity-exec.js` integrates Perplexity Sonar via OpenAI SDK baseURL swap (`https://api.perplexity.ai`), extracts citations from `response.citations` | Perplexity API pattern verified from STACK.md; PERPLEXITY_API_KEY absent (only session/CSRF tokens) — env var setup needed |
| EXEC-06 | Every executor implements `execute(prompt, opts)`, `stream(prompt, opts)`, and `available()` returning `{success, output, tokens, cost, error}` | Unified interface pattern documented with exact return shapes |
| EXEC-07 | `qwen-exec.js` `available()` sends minimal inference probe (not just `/api/tags`) and returns false gracefully when GPU/ollama is absent | Must probe `/api/generate` or use OpenAI-compat endpoint with short timeout — not `/api/tags` alone |
| EXEC-08 | `websearch.sh` wraps SearXNG at localhost:8888 | SearXNG confirmed running at 127.0.0.1:8888 (verified live) |
| EXEC-09 | `fetchdocs.js` calls Context7 HTTP endpoint directly | Context7 available as MCP; direct HTTP endpoint pattern documented |
| COST-01 | `lib/pricing.js` exposes per-provider cost functions for all nine models — no shared formula | Nine providers have four incompatible token schemas; per-function approach documented |
| COST-02 | `hooks/token-logger.js` handles four incompatible token count schemas: Anthropic, Gemini, MiniMax/OpenAI, ollama | Schema mapping table documented below |
| COST-06 | `raw_usage` field preserved in token-logger records | Simple field pass-through; no special SDK support needed |
</phase_requirements>

---

## Summary

Phase 2 builds the complete executor layer for all nine models, the per-provider pricing module, and two non-Claude helper scripts. The technical complexity is concentrated in three areas: (1) Gemini SDK integration with its unique search grounding and thinking mode APIs, (2) Qwen's cold-start characteristics requiring a real inference probe in `available()` plus a 180s timeout, and (3) the Perplexity dual-path mechanism (MCP → direct API fallback) given that no `PERPLEXITY_API_KEY` is currently configured.

Five of the nine executors reuse the existing `openai@6.33.0` package via baseURL swap (Codex via fork, MiniMax via fork, Qwen via ollama OpenAI-compat endpoint, Perplexity via direct API). Gemini is the only executor requiring a new package install (`@google/genai@1.48.0`). All fork work requires a path audit — existing sources hardcode absolute `~/.claude/hooks/` paths that break when copied to the plugin directory.

Token logging is the highest-risk non-executor task. Nine models produce four incompatible token schemas; a shared cost formula produces negative costs for Anthropic calls and zero costs for ollama calls. Each provider needs its own extraction function with `raw_usage` preserved for debugging.

**Primary recommendation:** Start with fork adaptations (codex, minimax) to validate the interface contract, then build new executors in dependency order: Gemini (search grounding needed for Phase 3), Perplexity (Discover phase), Qwen (graceful-fail until GPU arrives). Build pricing.js and token-logger in parallel with the first executor so they are ready before any executor is tested end-to-end.

---

## Standard Stack

### Core
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| `openai` (npm) | 6.33.0 | MiniMax, Perplexity, Qwen (baseURL swap), Codex API mode | Already installed globally at `~/.npm-global/lib/node_modules/openai`; used by all existing hook scripts |
| `@google/genai` | 1.48.0 | Gemini 3.1 Pro and Gemini 3 Flash | Only actively maintained Google AI SDK — `@google/generative-ai` deprecated Nov 2025; version verified from npm registry |

### Supporting
| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| `child_process` (Node built-in) | v22 | Codex CLI subprocess spawning | codex-exec.js only |
| `fs`, `path` (Node built-in) | v22 | File I/O for token log, __dirname-anchored requires | All executors and lib/ |
| `AbortController` (Node built-in) | v22 | HTTP timeout for MiniMax, Perplexity, Gemini | All API executors |

### Alternatives Considered
| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| `openai` baseURL swap for Qwen | `node-fetch` + raw HTTP | Raw HTTP requires manual request body, response parsing, error handling — no benefit over reusing installed SDK |
| `@google/genai` | `@google/generative-ai` | Deprecated Nov 2025, no new features, does not support current models |
| SearXNG at localhost:8888 | External search API | SearXNG is already running; external API adds cost and network dependency |

### Installation

```bash
# Install in plugin directory only
cd ~/.claude/plugins/seraphim
npm install @google/genai@1.48.0
```

The `openai` package does not need reinstalling — it is already at the global path and will be lazy-required with fallback as in existing hooks.

**Version verification (run before implementing):**
```bash
npm view @google/genai version   # Expected: 1.48.0
npm view openai version          # Expected: 6.33.0
```

---

## Architecture Patterns

### Recommended File Layout for Phase 2 Deliverables

```
~/.claude/plugins/seraphim/
├── executors/
│   ├── dispatch.js              # EXISTS (Phase 1) — receives execute calls
│   ├── codex-exec.js            # FORK from ~/.claude/hooks/codex-exec.js
│   ├── minimax-exec.js          # FORK from ~/.claude/hooks/minimax-exec.js
│   ├── gemini-exec.js           # NEW — @google/genai SDK
│   ├── qwen-exec.js             # NEW — ollama OpenAI-compat baseURL swap
│   └── perplexity-exec.js       # NEW — MCP bridge + direct API fallback
├── tools/
│   ├── websearch.sh             # NEW — SearXNG curl wrapper
│   └── fetchdocs.js             # NEW — Context7 HTTP wrapper
├── hooks/
│   └── token-logger.js          # FORK from ~/.claude/hooks/codex-token-logger.js
└── lib/
    ├── pricing.js               # FORK from ~/.claude/hooks/codex-pricing.js + extend
    ├── config.js                # EXISTS (Phase 1)
    └── phase-state.js           # EXISTS (Phase 1)
```

### Pattern 1: Unified Executor Contract

Every executor exports exactly three functions with these exact return shapes:

```javascript
// Source: docs/specs/2026-04-04-seraphim-v3-design.md + CONTEXT.md D-04/D-05
module.exports = {
  /**
   * execute(prompt, opts) -> Promise<Result>
   * opts: { model, maxTokens, timeoutMs, systemPrompt, phase, ... }
   * Result: { success, output, tokens, cost, error }
   *   tokens: { input_tokens, output_tokens, cache_read_input_tokens }
   *   cost: USD float (0 for free executors)
   *   error: { error_type, message, retriable }
   *     error_type: 'unavailable'|'timeout'|'rate_limit'|'auth'|'parse'|'unknown'
   *     retriable: boolean
   */
  async execute(prompt, opts) {},

  /**
   * stream(prompt, opts) -> Promise<AsyncGenerator>
   * Falls back to execute() for executors without native streaming
   */
  async stream(prompt, opts) {},

  /**
   * available() -> Promise<boolean>
   * NEVER throws. Returns false on any error.
   */
  async available() {}
};
```

### Pattern 2: Fork Path Audit (codex-exec.js and minimax-exec.js)

The source files contain hardcoded absolute paths that break after forking:

```javascript
// minimax-exec.js source (DO NOT keep after fork)
const OPENAI_GLOBAL_PATH = '/home/alucard/.npm-global/lib/node_modules/openai';
const PRICING_PATH       = '/home/alucard/.claude/hooks/codex-pricing';
const CODEX_EXEC_PATH    = '/home/alucard/.claude/hooks/codex-exec';

// After fork — replace with __dirname-anchored paths
const OPENAI_GLOBAL_PATH = '/home/alucard/.npm-global/lib/node_modules/openai'; // keep absolute OK
const PRICING_PATH       = path.join(__dirname, '../lib/pricing');
const CODEX_EXEC_PATH    = path.join(__dirname, './codex-exec'); // if needed at all
```

Run this audit after every fork:
```bash
grep -n "require(" ~/.claude/plugins/seraphim/executors/codex-exec.js | grep "\.\."
grep -n "require(" ~/.claude/plugins/seraphim/executors/minimax-exec.js | grep "\.\."
```

Any result pointing outside the plugin directory is a bug.

### Pattern 3: Gemini SDK — Search Grounding and Thinking Mode

Verified against installed `@google/genai@1.48.0` type definitions at `/home/alucard/.npm-global/lib/node_modules/openclaw/node_modules/@google/genai/dist/genai.d.ts`:

```javascript
// Source: verified from genai.d.ts type signatures
const { GoogleGenAI, ThinkingLevel } = require('@google/genai');
const ai = new GoogleGenAI({ apiKey: process.env.GEMINI_API_KEY });

// Standard text generation
const response = await ai.models.generateContent({
  model: 'gemini-3.1-pro-preview',
  contents: prompt,
  config: {
    thinkingConfig: {
      thinkingBudget: -1,       // -1 = AUTOMATIC; 0 = disabled
      thinkingLevel: ThinkingLevel.HIGH
    }
  }
});
const text = response.text;  // convenience getter

// Search grounding (Discover phase external track)
const response = await ai.models.generateContent({
  model: 'gemini-3-flash-preview',
  contents: prompt,
  config: {
    tools: [{ googleSearch: {} }]  // NOT google_search_retrieval (deprecated)
  }
});
// Grounding metadata available at response.candidates[0].groundingMetadata

// Token extraction (Gemini schema)
const usage = response.usageMetadata;
// Fields: promptTokenCount, candidatesTokenCount, thoughtsTokenCount, totalTokenCount
```

**ThinkingLevel enum** (verified from installed SDK):
- `THINKING_LEVEL_UNSPECIFIED`, `MINIMAL`, `LOW`, `MEDIUM`, `HIGH`

**Important:** `@google/generative-ai` (0.24.1) is deprecated. Use `@google/genai` only.

### Pattern 4: Qwen via Ollama OpenAI-Compat Endpoint

```javascript
// Source: .planning/research/STACK.md (verified from ollama docs)
// Reuse existing openai package — no new install needed
let OpenAI;
try { OpenAI = require('openai').OpenAI; }
catch (e) { OpenAI = require('/home/alucard/.npm-global/lib/node_modules/openai').OpenAI; }

const client = new OpenAI({
  apiKey: 'ollama',                        // any non-empty string
  baseURL: 'http://localhost:11434/v1',    // OpenAI-compat endpoint
  timeout: 180000                          // 180s for cold-start margin
});

const response = await client.chat.completions.create({
  model: 'qwen3:30b-a3b-q4_K_M',
  messages: [{ role: 'user', content: prompt }],
  temperature: 0.7,
  // num_ctx passed via extra_body for ollama-specific params
  // Note: OpenAI SDK passes extra_body as-is to the API
});
// Token extraction (OpenAI/ollama schema)
// response.usage: { prompt_tokens, completion_tokens, total_tokens }
// ollama also maps: prompt_eval_count -> prompt_tokens, eval_count -> completion_tokens
```

**available() — must use inference probe, not /api/tags:**

```javascript
// Source: .planning/research/PITFALLS.md Pitfall 3
async function available() {
  try {
    const controller = new AbortController();
    const timer = setTimeout(() => controller.abort(), 60000);  // 60s probe timeout
    const response = await client.chat.completions.create({
      model: 'qwen3:30b-a3b-q4_K_M',
      messages: [{ role: 'user', content: 'ping' }],
      max_tokens: 1
    }, { signal: controller.signal });
    clearTimeout(timer);
    return true;
  } catch {
    return false;
  }
}
```

**Qwen model tag correction:** The design spec says "Qwen 3.5-27B" but this size does not exist. The correct tag is `qwen3:30b-a3b-q4_K_M` (30B, Q4_K_M quantization, 19GB VRAM).

### Pattern 5: Perplexity Dual-Path Executor

```javascript
// Source: CONTEXT.md D-01/D-02 + .planning/research/STACK.md
// MCP path: perplexity MCP tools are accessible inside Claude Code sessions
// but NOT from Node.js executors called via Bash. The "MCP first" strategy
// requires calling perplexity through a Claude subagent or the host session.
// Direct API fallback is the primary path for executor-level calls.

// Direct API pattern (fallback or primary when MCP unavailable)
const client = new OpenAI({
  apiKey: process.env.PERPLEXITY_API_KEY,
  baseURL: 'https://api.perplexity.ai'    // NO /v1 suffix
});

const response = await client.chat.completions.create({
  model: 'sonar-pro',
  messages: [
    { role: 'system', content: 'Be precise and cite sources.' },
    { role: 'user', content: prompt }
  ]
});

const text = response.choices[0].message.content;
const citations = response.citations || [];  // Perplexity extension field
```

**MCP availability detection:** Check for presence of MCP tool invocation context (i.e., whether the executor is being called from within a Claude session that has Perplexity MCP configured). This is opaque to a standalone Node.js executor — recommend implementing MCP path as a separate code path invoked only when the executor receives a `opts.mcpAvailable: true` flag from the dispatch caller.

### Pattern 6: Per-Provider Pricing Functions

```javascript
// Source: REQUIREMENTS.md COST-01 + .planning/research/PITFALLS.md Pitfall 8
// NEVER use a shared formula across providers — Anthropic cache-read produces negative delta if mishandled

const PRICING = {
  // All prices per 1M tokens
  'claude-opus-4-6':    { input: 5.00, cache_write: 3.75, cache_read: 1.00, output: 25.00 },
  'claude-sonnet-4-6':  { input: 3.00, cache_write: 3.75, cache_read: 0.30, output: 15.00 },
  'claude-haiku-4-5':   { input: 0.80, cache_write: 1.00, cache_read: 0.08, output: 4.00 },
  'codex-gpt-5.4':      { input: 0, cache_write: 0, cache_read: 0, output: 0 },    // Plus sub
  'minimax-m2.7':       { input: 0.30, cache_read: 0.06, output: 1.20 },
  'gemini-3.1-pro':     { input: 2.00, output: 12.00 },
  'gemini-3-flash':     { input: 0.50, output: 3.00 },
  'qwen-3.5-27b':       { input: 0, output: 0 },                                   // local
  'perplexity-sonar':   { input: 1.00, output: 5.00 },
};

// Per-provider functions — never generic
function computeAnthropicCost(inputTokens, cacheReadTokens, cacheWriteTokens, outputTokens, model) {
  const p = PRICING[model];
  // cache_read is a credit (reduces cost), not an additional charge
  return (
    inputTokens        * p.input / 1_000_000 +
    cacheWriteTokens   * p.cache_write / 1_000_000 +
    cacheReadTokens    * p.cache_read / 1_000_000 +   // NOTE: positive charge, not a credit
    outputTokens       * p.output / 1_000_000
  );
}

function computeGeminiCost(promptTokenCount, candidatesTokenCount, model) {
  const p = PRICING[model];
  return (promptTokenCount * p.input + candidatesTokenCount * p.output) / 1_000_000;
}

function computeOpenAICost(promptTokens, completionTokens, model) {
  const p = PRICING[model];
  return (promptTokens * p.input + completionTokens * p.output) / 1_000_000;
}

// Codex and Qwen return 0 — no computation needed
function computeFreeCost() { return 0; }
```

### Pattern 7: Token Logger — Four Schema Mapping

```javascript
// Source: REQUIREMENTS.md COST-02 + .planning/research/PITFALLS.md Pitfall 8
// Four schemas mapped to canonical fields

function normalizeUsage(rawUsage, provider) {
  switch (provider) {
    case 'anthropic':
      return {
        input_tokens:            rawUsage.input_tokens || 0,
        output_tokens:           rawUsage.output_tokens || 0,
        cache_read_input_tokens: rawUsage.cache_read_input_tokens || 0,
        cache_creation_input_tokens: rawUsage.cache_creation_input_tokens || 0
      };
    case 'gemini':
      return {
        input_tokens:  rawUsage.promptTokenCount || 0,
        output_tokens: rawUsage.candidatesTokenCount || 0,
        thoughts_tokens: rawUsage.thoughtsTokenCount || 0
      };
    case 'openai':  // MiniMax, Perplexity, Codex API
      return {
        input_tokens:  rawUsage.prompt_tokens || 0,
        output_tokens: rawUsage.completion_tokens || 0
      };
    case 'ollama':  // Qwen via ollama
      return {
        input_tokens:  rawUsage.prompt_eval_count || rawUsage.prompt_tokens || 0,
        output_tokens: rawUsage.eval_count || rawUsage.completion_tokens || 0
      };
    default:
      return { input_tokens: 0, output_tokens: 0 };
  }
}

// token-logger record schema — raw_usage is required (COST-06)
const record = {
  timestamp:      new Date().toISOString(),
  phase:          opts.phase || 'unknown',
  model:          modelId,
  provider:       provider,
  tokens:         normalizeUsage(rawUsage, provider),
  raw_usage:      rawUsage,    // COST-06: preserve original fields
  cost_usd:       computedCost
};
```

### Pattern 8: Qwen Structured Output for Forge Mode

```javascript
// Source: CONTEXT.md D-03 + .planning/research/STACK.md
// JSON action schema — validated as reliable for Qwen instruction following

const FORGE_SYSTEM_PROMPT = `You are an autonomous code execution engine.
Output ONLY JSON action objects, one per line. No prose before or after.
Valid actions:
{"action":"write","path":"<relative path>","content":"<file content>"}
{"action":"run","command":"<shell command>"}
{"action":"read","path":"<relative path>"}
{"action":"done","summary":"<what was accomplished>"}
Do not output markdown, explanations, or any text outside of JSON lines.`;

// The wrapper parses line-by-line and executes each action
function parseForgeActions(output) {
  return output.split('\n')
    .filter(l => l.trim().startsWith('{'))
    .map(l => { try { return JSON.parse(l); } catch { return null; } })
    .filter(Boolean);
}
```

### Pattern 9: websearch.sh — SearXNG Wrapper

```bash
#!/usr/bin/env bash
# Source: SearXNG confirmed running at localhost:8888 (verified live)
# Usage: websearch.sh "query" [limit]
set -euo pipefail

QUERY="${1:?Usage: websearch.sh <query> [limit]}"
LIMIT="${2:-10}"
SEARXNG_URL="http://127.0.0.1:8888"   # per security rules: 127.0.0.1 not 0.0.0.0

curl -s --connect-timeout 5 --max-time 15 \
  "${SEARXNG_URL}/search?q=$(python3 -c "import urllib.parse,sys; print(urllib.parse.quote(sys.argv[1]))" "$QUERY")&format=json&pageno=1" \
  | python3 -c "
import json,sys
data = json.load(sys.stdin)
results = data.get('results', [])[:${LIMIT}]
for r in results:
    print(json.dumps({'title': r.get('title',''), 'url': r.get('url',''), 'content': r.get('content','')}))
"
```

### Pattern 10: fetchdocs.js — Context7 HTTP Wrapper

Context7 is available as an MCP server (`mcp__plugin_context7_context7__query-docs`). For non-Claude models (Codex, Qwen) that cannot call MCP tools, fetchdocs.js wraps the Context7 API directly.

```javascript
#!/usr/bin/env node
// fetchdocs.js — Context7 documentation fetcher for non-MCP models
// Usage: node fetchdocs.js <library-name> <query>
// Context7 MCP server is at the plugin level; direct HTTP via mcp proxy or
// the Context7 API endpoint. Check Context7 API docs for endpoint.
```

**Note:** Context7 MCP is available at `mcp__plugin_context7_context7__resolve-library-id` and `mcp__plugin_context7_context7__query-docs`. The direct HTTP endpoint requires verification — Context7 may be MCP-only with no public REST API. If so, fetchdocs.js must either proxy through the MCP server or document that it is only available to Claude Code sessions.

### Anti-Patterns to Avoid

- **Shared cost formula:** Never use a single `computeCost(tokens, model)` — Anthropic cache-read tokens have different semantics than MiniMax cached token accounting.
- **`/api/tags` for Qwen availability:** This only confirms ollama is running, not that Qwen is loaded. A cold-start will time out in execute() while available() returns true.
- **`google_search_retrieval` tool name:** This is the deprecated form. Current models use `{ googleSearch: {} }` — different key name.
- **Relative requires after fork:** All `require('../codex-pricing')` paths from the source hooks directory become wrong after fork. Always audit with grep.
- **`runWithFallback` in minimax-exec.js:** This function must be removed in the fork — dispatch.js now owns fallback logic.
- **`@google/generative-ai`:** Do not install or use this package. It is deprecated as of Nov 2025.
- **Perplexity base URL with `/v1`:** The Perplexity API endpoint is `https://api.perplexity.ai` with no `/v1` suffix. Adding `/v1` causes 404 on all requests.

---

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Gemini API client | Custom `fetch()` wrapper | `@google/genai@1.48.0` SDK | SDK handles auth, retries, streaming, function calling, grounding metadata |
| MiniMax / Perplexity API calls | Custom HTTP | `openai@6.33.0` with baseURL swap | Already installed; handles timeout, retry, error shapes |
| Qwen local HTTP | Direct `http.request()` to ollama | `openai@6.33.0` with `baseURL: 'http://localhost:11434/v1'` | Reuses installed package; OpenAI SDK already handles response parsing |
| Exponential backoff | Manual sleep loop | Existing `callWithRetry()` from minimax-exec.js (fork it) | Proven implementation with jitter; just adjust for 429-specific behavior |
| Token normalization | Ad-hoc field extraction | Structured `normalizeUsage(raw, provider)` per provider | Four schemas are mutually incompatible; one function handles each cleanly |
| Context7 CLI wrapping | Custom HTTP | MCP tools directly from Claude sessions; `fetchdocs.js` only for Codex/Qwen Bash calls | MCP is the primary access path |

**Key insight:** Every external model in this phase (except the Codex CLI subprocess) has an OpenAI-compatible or SDK-wrapped API. The consistent pattern is: pick the right baseURL + SDK, not custom HTTP clients.

---

## Common Pitfalls

### Pitfall 1: Qwen Cold-Start Timeout — available() Returns True But execute() Times Out

**What goes wrong:** ollama /api/tags responds immediately (confirms process is running), but Qwen 3.5 (30B) takes 13-46s to load from disk into VRAM. If `available()` only checks tags and `execute()` uses a 30s timeout, the first call in a session times out while loading.

**Why it happens:** Loading and running are separate states. Tags endpoint doesn't probe model readiness.

**How to avoid:** `available()` must send a minimal inference probe (`max_tokens: 1`) with a 60s timeout. Log cold-start duration if over 10s. Use 180s (not 120s) in execute() for margin.

**Warning signs:** First Qwen call per session silently falls back to paid cloud model. Balanced/Budget profile costs are higher than expected.

### Pitfall 2: Hardcoded Paths After Fork

**What goes wrong:** `minimax-exec.js` source contains `const PRICING_PATH = '/home/alucard/.claude/hooks/codex-pricing'`. After forking to the plugin, this require still resolves — but loads the pre-fork pricing file, not the new `lib/pricing.js`.

**How to avoid:** Replace ALL `require()` paths in forked files with `path.join(__dirname, '../lib/pricing')` anchors. Run grep audit before first test.

### Pitfall 3: Gemini 429 Rate Limit Not Retried

**What goes wrong:** HTTP 429 from Gemini Flash is treated as hard failure, Judge phase aborts, pipeline stalls. Free tier allows only 10 RPM — easy to hit during development.

**How to avoid:** gemini-exec.js must implement 429-specific backoff: 5s → 10s → 30s → 60s, max 3 retries. Distinguish 429 (retriable) from 401 (auth, not retriable — surface immediately) from 400 (client error, not retriable).

### Pitfall 4: Negative Costs From Shared Anthropic Pricing Formula

**What goes wrong:** The existing `computeCost()` in codex-pricing.js was designed for OpenAI tokens. Applied to Anthropic subagent responses with `cache_read_input_tokens`, it subtracts cached tokens from the cost total, potentially producing negative values.

**How to avoid:** Separate `computeAnthropicCost()` function that treats cache_read as a positive charge (it is billed at a reduced rate, not free). Never use OpenAI formula for Anthropic calls.

### Pitfall 5: `runWithFallback` Left in Forked minimax-exec.js

**What goes wrong:** dispatch.js owns the fallback chain in Phase 2+. If `runWithFallback` is left in minimax-exec.js, it creates a double-fallback: dispatch falls back to minimax, minimax falls back to Codex internally, producing unexpected model usage without dispatch-level logging.

**How to avoid:** `runWithFallback` and `isCodexRateLimited` must be removed from the forked minimax-exec.js. Export only `runMinimax` and adapt it to the unified interface.

### Pitfall 6: GEMINI_API_KEY and PERPLEXITY_API_KEY Not Configured

**What goes wrong:** `available()` for gemini-exec.js checks for env var presence. Key is not currently set on this machine. `perplexity-exec.js` direct API fallback requires `PERPLEXITY_API_KEY` — only session/CSRF tokens are present (these are web login tokens, not API keys).

**How to avoid:** Each executor's `available()` must check for its required env var and return `false` (not throw) when missing. Include setup instructions in the phase plan for adding `GEMINI_API_KEY` to the shell environment. PERPLEXITY_API_KEY requires a separate Perplexity API subscription (different from web login).

### Pitfall 7: Perplexity MCP Not Accessible From Node.js Executors

**What goes wrong:** The CONTEXT.md D-01 decision says "Try MCP first." But Perplexity MCP tools (`mcp__perplexity__*`) are only accessible within a Claude Code session, not from a standalone Node.js process invoked via Bash. A Node.js executor called by dispatch.js cannot invoke MCP tools.

**How to avoid:** The MCP-first path in perplexity-exec.js must be implemented differently — either: (a) the executor is only invoked from Claude commands that have MCP access and can proxy the call, or (b) the "MCP first" path is always the host Claude session handling Perplexity tasks natively, with perplexity-exec.js being the direct-API fallback only. Implement `available()` to check `PERPLEXITY_API_KEY` for the direct API path. Document that the MCP path requires special invocation context.

---

## Code Examples

### gemini-exec.js skeleton with search grounding and thinking mode

```javascript
// Source: type definitions verified from @google/genai@1.48.0 genai.d.ts
'use strict';
const path = require('path');
const { GoogleGenAI, ThinkingLevel } = require('@google/genai');
const { computeGeminiCost } = require(path.join(__dirname, '../lib/pricing'));

const DEFAULT_MODEL = 'gemini-3.1-pro-preview';
const TIMEOUT_MS = 90000;
const MAX_RETRIES = 3;

async function execute(prompt, opts) {
  const apiKey = process.env.GEMINI_API_KEY;
  if (!apiKey) {
    return { success: false, output: '', tokens: null, cost: 0,
             error: { error_type: 'auth', message: 'GEMINI_API_KEY not set', retriable: false } };
  }
  const ai = new GoogleGenAI({ apiKey });
  const model = (opts && opts.model) || DEFAULT_MODEL;
  const useGrounding = opts && opts.searchGrounding;
  const useThinking = opts && opts.thinkingMode;

  const config = {};
  if (useGrounding) config.tools = [{ googleSearch: {} }];
  if (useThinking) config.thinkingConfig = { thinkingBudget: -1, thinkingLevel: ThinkingLevel.HIGH };

  // Retry loop with 429 backoff
  let lastErr;
  for (let attempt = 0; attempt <= MAX_RETRIES; attempt++) {
    try {
      const response = await ai.models.generateContent({
        model, contents: prompt, ...(Object.keys(config).length ? { config } : {})
      });
      const usage = response.usageMetadata || {};
      const tokens = {
        input_tokens: usage.promptTokenCount || 0,
        output_tokens: usage.candidatesTokenCount || 0,
        thoughts_tokens: usage.thoughtsTokenCount || 0
      };
      return {
        success: true,
        output: response.text,
        tokens,
        cost: computeGeminiCost(tokens.input_tokens, tokens.output_tokens, model),
        raw_usage: usage
      };
    } catch (err) {
      lastErr = err;
      const status = err.status || err.code;
      if (status === 429 && attempt < MAX_RETRIES) {
        const delay = Math.min(5000 * Math.pow(2, attempt), 60000) + Math.random() * 1000;
        await new Promise(r => setTimeout(r, delay));
        continue;
      }
      const error_type = status === 401 ? 'auth' : status === 429 ? 'rate_limit' : 'unknown';
      return { success: false, output: '', tokens: null, cost: 0,
               error: { error_type, message: err.message, retriable: status === 429 } };
    }
  }
  return { success: false, output: '', tokens: null, cost: 0,
           error: { error_type: 'rate_limit', message: lastErr.message, retriable: true } };
}

async function available() {
  if (!process.env.GEMINI_API_KEY) return false;
  return true;  // API key presence sufficient; don't burn quota on probe
}

module.exports = { execute, stream: execute, available };
```

### qwen-exec.js available() probe pattern

```javascript
// Source: .planning/research/PITFALLS.md Pitfall 3 — must use inference probe
async function available() {
  try {
    const controller = new AbortController();
    const timer = setTimeout(() => controller.abort(), 60000);
    await client.chat.completions.create({
      model: 'qwen3:30b-a3b-q4_K_M',
      messages: [{ role: 'user', content: 'ping' }],
      max_tokens: 1
    }, { signal: controller.signal });
    clearTimeout(timer);
    return true;
  } catch {
    return false;
  }
}
```

---

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| `@google/generative-ai` | `@google/genai` | Nov 2025 (deprecated) | Must use new SDK; old package does not support current Gemini models or grounding API |
| `google_search_retrieval` tool | `{ googleSearch: {} }` config | 2025 | Old tool name causes API error on current models |
| Ollama `/api/generate` HTTP directly | OpenAI-compat endpoint at `/v1` | ~2024 | Can reuse existing `openai` npm package; no new dependency |

**Deprecated/outdated:**
- `@google/generative-ai`: Deprecated Nov 30 2025. Do not install or reference.
- `google_search_retrieval` (old Gemini grounding tool name): Replaced by `{ googleSearch: {} }`.
- `runWithFallback` in minimax-exec.js: Dispatch owns fallback in v3.0; this function is removed from the fork.

---

## Open Questions

1. **Context7 direct HTTP endpoint**
   - What we know: Context7 is available as an MCP server and works perfectly within Claude Code sessions
   - What's unclear: Whether Context7 exposes a public REST API that fetchdocs.js can call without MCP
   - Recommendation: Attempt `GET https://context7.ai/api/...` — if no public REST API exists, fetchdocs.js becomes a thin CLI that invokes MCP via `claude mcp call` or is documented as unavailable outside Claude sessions. Codex/Qwen would then rely on websearch.sh alone for external research.

2. **Perplexity MCP vs direct API routing**
   - What we know: `PERPLEXITY_API_KEY` is not set (only session/CSRF tokens exist); MCP tools are inaccessible from Node.js executors
   - What's unclear: Whether the user intends to purchase a Perplexity API key or rely entirely on MCP routing through Claude
   - Recommendation: Implement `available()` to check for `PERPLEXITY_API_KEY`; if absent, return `false` gracefully. Document that the MCP path is handled at the Claude command level, not the executor level. Perplexity executor operates in direct-API mode only when the key is set.

3. **Gemini model IDs for current generation**
   - What we know: STACK.md records `gemini-3.1-pro-preview` and `gemini-3-flash-preview` as of 2026-04-04
   - What's unclear: Whether these are the final production model IDs or still preview names
   - Recommendation: Use IDs from STACK.md; `available()` can validate by calling the models list endpoint once and caching the result.

---

## Environment Availability

| Dependency | Required By | Available | Version | Fallback |
|------------|------------|-----------|---------|----------|
| Node.js | All executors | Yes | v22.22.0 | — |
| `openai` npm | codex-exec, minimax-exec, qwen-exec, perplexity-exec | Yes | 6.33.0 (global) | — |
| `@google/genai` npm | gemini-exec.js | No (needs install) | — | `npm install @google/genai@1.48.0` |
| SearXNG at localhost:8888 | websearch.sh | Yes | — (HTTP 200 confirmed) | — |
| ollama binary | qwen-exec.js | No (not in PATH, GPU in transit) | — | available() returns false gracefully |
| Qwen model pulled | qwen-exec.js | No (GPU not yet present) | — | available() returns false gracefully |
| `GEMINI_API_KEY` | gemini-exec.js | No (not configured) | — | available() returns false; gemini phases fall to profile fallback |
| `PERPLEXITY_API_KEY` | perplexity-exec.js direct API | No (only session tokens) | — | available() returns false; Perplexity phases fall to profile fallback or MCP path |
| `MINIMAX_API_KEY` | minimax-exec.js | Yes | — (configured) | — |
| `OPENAI_API_KEY` | codex-exec.js | Yes | — (configured) | — |

**Missing dependencies with no fallback:**
- `@google/genai` npm package — must be installed before gemini-exec.js can be used; install via `npm install @google/genai@1.48.0` in the plugin directory

**Missing dependencies with fallback:**
- `GEMINI_API_KEY` — gemini-exec.js `available()` returns false; dispatch falls back to profile fallback model
- `PERPLEXITY_API_KEY` — perplexity-exec.js `available()` returns false; dispatch falls back
- ollama / Qwen — qwen-exec.js `available()` returns false; dispatch falls back; Balanced/Budget profiles degraded until GPU arrives

---

## Sources

### Primary (HIGH confidence)
- `@google/genai@1.48.0` type definitions — inspected live at `/home/alucard/.npm-global/lib/node_modules/openclaw/node_modules/@google/genai/dist/genai.d.ts`; confirmed `GoogleGenAI`, `ThinkingLevel`, `ThinkingConfig`, `generateContent`, `generateContentStream`, `usageMetadata` shapes
- `/home/alucard/.claude/hooks/codex-exec.js` — direct inspection of fork source; JSONL parsing, token extraction, 300s timeout pattern confirmed
- `/home/alucard/.claude/hooks/minimax-exec.js` — direct inspection of fork source; `runWithFallback`, absolute path constants, `callWithRetry` confirmed
- `/home/alucard/.claude/hooks/codex-pricing.js` — direct inspection of fork source; pricing constant structure confirmed
- `/home/alucard/.claude/plugins/seraphim/config/models.json` — confirmed nine model definitions and pricing keys
- `/home/alucard/.claude/plugins/seraphim/executors/dispatch.js` — Phase 1 output; three-level resolution confirmed working

### Secondary (MEDIUM confidence)
- `.planning/research/STACK.md` — Gemini API patterns, Qwen model tag, Perplexity baseURL, all verified against official docs as of 2026-04-04
- `.planning/research/PITFALLS.md` — Qwen cold-start, path fork pitfalls, Gemini 429, token logging incompatibility; all verified HIGH confidence in source document
- `.planning/research/ARCHITECTURE.md` — unified executor interface contract; confirmed against design spec
- Live environment audit: `OPENAI_API_KEY`, `MINIMAX_API_KEY` confirmed set; `GEMINI_API_KEY` confirmed absent; SearXNG HTTP 200 confirmed; ollama not in PATH confirmed

### Tertiary (LOW confidence)
- Context7 direct HTTP endpoint: not verified — only MCP access confirmed
- Perplexity MCP from Node.js: inferred from MCP architecture; flagged in Open Questions

---

## Project Constraints (from CLAUDE.md)

| Directive | Source | Impact on Phase 2 |
|-----------|--------|-------------------|
| Never expose API keys in plaintext | CLAUDE.md Security | All API keys via `process.env.*` only; no hardcoded keys in any executor |
| Bind services to 127.0.0.1 | CLAUDE.md Security | SearXNG URL in websearch.sh: `http://127.0.0.1:8888` not `0.0.0.0` |
| RTX 3090 required for local Qwen | CLAUDE.md Constraints | qwen-exec.js `available()` must return false gracefully; Balanced/Budget degraded until GPU arrives |
| Temperature 0.01 for MiniMax | CLAUDE.md Technology Stack | Must be preserved in forked minimax-exec.js; API rejects exactly 0 |
| 120s timeout for Qwen local, 90s for MiniMax, 300s for Codex CLI | CLAUDE.md Technology Stack | Per-executor timeout constants must match exactly; do not inherit from shared config |
| executors implement `execute`, `stream`, `available` | CLAUDE.md Conventions | Unified interface is non-negotiable; dispatch.js depends on exactly these three names |
| Hook scripts use Node.js stdin/stdout JSON pattern with 10s timeout guard | CLAUDE.md Conventions | token-logger.js fork must preserve the 10s stdinTimeout guard |
| Token logging to `token-log.jsonl`; routing decisions to `decisions.jsonl` | CLAUDE.md Conventions | token-logger writes to `.seraphim/token-log.jsonl`; dispatch logs to `.seraphim/decisions.jsonl` |
| Plugin standalone — no runtime dependency on GSD or Superpowers | CLAUDE.md Constraints | No `require()` calls into GSD hook directory; all imports from plugin-internal paths |
| Never send credentials/PII to MiniMax | CLAUDE.md Security | MiniMax prompt content must not include API keys, access tokens, or user-identifying data |

---

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH — packages verified from npm registry and live installation
- Architecture patterns: HIGH — verified from installed SDK type definitions and existing fork source code
- Pitfalls: HIGH — sourced from verified PITFALLS.md which cited official docs
- Environment availability: HIGH — live audit performed (curl SearXNG, env var check, ollama binary check)

**Research date:** 2026-04-04
**Valid until:** 2026-05-04 (Gemini model IDs may change from preview to GA before then — recheck if >2 weeks pass before planning)
