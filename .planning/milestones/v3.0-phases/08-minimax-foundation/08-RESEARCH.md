# Phase 8: MiniMax Foundation - Research

**Researched:** 2026-04-03
**Domain:** MiniMax M-2.7 API integration, Node.js shared module patterns, pricing correction, migration scripting
**Confidence:** HIGH

---

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions

**Module architecture:**
- D-01: Standalone `minimax-exec.js` module, separate from `codex-exec.js`. Clean separation of concerns — each model provider has its own module.
- D-02: Exports `runMinimax(prompt, opts)` as the primary function. Uses the existing `openai` npm package (v6.33.0, already installed) with `baseURL: https://api.minimax.io/v1`.
- D-03: Temperature set to `0.01` (MiniMax API rejects exactly `0`; range must be `(0.0, 1.0]`).
- D-04: Default `max_tokens: 2000` to control MiniMax's verbosity tax (~4x more output tokens than median models).
- D-05: Returns `{ success, text, tokens, cost }` shape — `cost` computed via `codex-pricing.js`.
- D-06: Also exports `runWithFallback(prompt, opts)` — tries Codex CLI first, falls back to MiniMax on rate-limit errors, then prompts user as last resort. Built into the foundation module from day one so Phases 9-12 can benefit from fallback resilience immediately.

**Pricing correction:**
- D-07: Fix `OPUS_PRICING` in `codex-pricing.js` from `{ input: 15.00, output: 75.00 }` (Opus 4.1) to `{ input: 5.00, output: 25.00 }` (Opus 4.6). Cached input from `3.75` to `1.25`.
- D-08: Add MiniMax pricing: `'minimax-m2.7': { input: 0.30, cached_input: 0.06, output: 1.20 }`.
- D-09: Full historical recalculation — write a migration script that re-processes all `token-log.jsonl` files across all projects, recalculates Opus baseline costs with corrected pricing, and regenerates `global.jsonl`. Dashboard savings percentages will change.

**Config & API key:**
- D-10: Separate `minimax` block in project `.claude/settings.json`, alongside the existing `codex` block. Not nested inside `codex`.
- D-11: Config fields: `enabled`, `model`, `api_key_env`, `base_url`, `max_tokens_default`, `tasks` (array of task types routed to MiniMax).
- D-12: Use Pay-As-You-Go API key (not Token Plan). Key never expires. $25 in credits already available.
- D-13: API key stored as `MINIMAX_API_KEY` environment variable. Never in plaintext files.

**Fallback wiring:**
- D-14: Fallback chain built into `minimax-exec.js` as `runWithFallback(prompt, opts)`.
- D-15: Codex rate-limit detection via: exit code + stderr "rate limit"/"quota"/"usage limit", HTTP 429 in JSONL output, timeout with no output, `rate_limit_pct >= 95`.
- D-16: On Codex failure: log the failure reason to `token-log.jsonl`, auto-retry via MiniMax API, log MiniMax usage as `source: 'api-fallback'`. On MiniMax failure: prompt user with both error messages and options (wait/retry, check key, skip task).
- D-17: Execution fallback is fail-closed (prompt user). Review fallback is fail-open (skip if both fail).

### Claude's Discretion
- Error handling and retry logic details (exponential backoff, jitter, max retries)
- Migration script implementation approach (batch vs streaming)
- Exact format of fallback log entries in token-log.jsonl
- Whether to add a `minimax-m2.7-highspeed` pricing entry alongside standard

### Deferred Ideas (OUT OF SCOPE)
- Token Plan evaluation — revisit after Phases 9-12 are running
- MiniMax `reasoning_split: true` support — needed for Phase 10 (adversarial review) but not Phase 8
- `minimax-m2.7-highspeed` variant ($0.60/$2.40) — add pricing entry if latency-sensitive use cases emerge
- OpenRouter as a secondary MiniMax endpoint — evaluate if MiniMax direct API has reliability issues
</user_constraints>

---

## Summary

Phase 8 lays the foundation for the three-model architecture by adding MiniMax M-2.7 as a provider module in the hook infrastructure. The work falls into four distinct tracks that can mostly be parallelized: (1) create `minimax-exec.js` as an OpenAI-SDK wrapper with `baseURL` swap, (2) extend `codex-pricing.js` with the MiniMax entry and fix the Opus 4.6 pricing error, (3) write a migration script to recalculate historical `opus_baseline_usd` values across all token logs, and (4) configure the environment and project settings block.

The integration requires zero new npm dependencies — the `openai` package (v6.33.0) already installed at `/home/alucard/.npm-global/lib/node_modules/openai` works as-is with a `baseURL` swap. The `MINIMAX_API_KEY` environment variable is not yet set on this machine; adding it to `~/.bashrc` is a prerequisite for the connectivity verification step. The existing token-log JSONL schema already supports a `source` field, so `source: 'api-fallback'` drops in without schema changes.

The pricing migration affects 82 records in `.planning/token-log.jsonl` and 215 records in `~/.claude/dashboard/global.jsonl`. The Opus baseline correction (from $15/$75 to $5/$25) will reduce reported savings percentages across the dashboard, but this is expected and correct. The migration script must be idempotent since the aggregator re-reads all token-log.jsonl files on every SessionStart.

**Primary recommendation:** Implement in three plans — Plan 1: pricing module changes + migration script; Plan 2: `minimax-exec.js` module; Plan 3: config/settings + connectivity verification.

---

## Standard Stack

### Core
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| `openai` npm package | 6.33.0 (installed at `/home/alucard/.npm-global/lib/node_modules/openai`) | OpenAI-compatible SDK call to MiniMax API | Already installed, used by `codex-exec.js` for `runGpt54MiniApi`. Same lazy-require pattern applies. |
| Node.js | v22.22.0 (installed) | Hook execution runtime | All existing hooks are Node.js; matching runtime eliminates friction |

### Supporting
| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| `child_process` (built-in) | Node.js built-in | Spawn Codex CLI for fallback detection | Already used in `codex-exec.js`; re-use the existing `runCodexExec` rather than spawning directly |

### Alternatives Considered
| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| OpenAI SDK with baseURL swap | Anthropic SDK (MiniMax also supports `/anthropic` endpoint) | Both work; OpenAI SDK matches existing hook pattern in `codex-exec.js`; locked by D-02 |
| Direct API (`fetch`) | OpenAI SDK | Fetch is lower overhead but loses type safety; SDK is already installed; locked by D-02 |

**Installation:** No new packages required. `openai` v6.33.0 is already installed globally.

**Version verification (confirmed 2026-04-03):**
```bash
node -e "console.log(require('/home/alucard/.npm-global/lib/node_modules/openai/package.json').version)"
# Output: 6.33.0
```

---

## Architecture Patterns

### Recommended Project Structure

```
~/.claude/hooks/
├── minimax-exec.js          # NEW — MiniMax provider module (mirrors codex-exec.js pattern)
├── codex-exec.js            # EXISTING — unchanged (shared codex+gpt-5.4-mini module)
├── codex-pricing.js         # MODIFY — add minimax-m2.7 entry, fix OPUS_PRICING
└── ...
~/.claude/hooks/
└── migrate-opus-pricing.js  # NEW — one-time migration script (idempotent)

.claude/settings.json        # MODIFY — add minimax block alongside codex block
~/.bashrc                    # MODIFY — add MINIMAX_API_KEY export
```

### Pattern 1: Lazy-Require OpenAI SDK (Mirror `runGpt54MiniApi`)

The established hook pattern for OpenAI SDK calls is a lazy require with fallback path. `minimax-exec.js` must follow this exactly — hooks run as child processes without `node_modules` context, so the package must be found at its global install path.

```javascript
// Source: ~/.claude/hooks/codex-exec.js (existing runGpt54MiniApi, lines 243-252)
'use strict';

const MINIMAX_OPENAI_PATH = '/home/alucard/.npm-global/lib/node_modules/openai';

function getOpenAIClient(baseURL, apiKey) {
  let OpenAI;
  try {
    try {
      OpenAI = require('openai').OpenAI;
    } catch (e) {
      OpenAI = require(MINIMAX_OPENAI_PATH).OpenAI;
    }
  } catch (e) {
    return { error: 'openai package not installed: ' + e.message };
  }
  return new OpenAI({ baseURL, apiKey });
}
```

**Confidence:** HIGH — verified from existing `codex-exec.js` source.

### Pattern 2: `runMinimax(prompt, opts)` Return Shape

The return shape `{ success, text, tokens, cost }` must be consistent with how downstream hooks (Phases 9-12) will consume it. Based on the `runGpt54MiniApi` pattern and the token-log JSONL schema:

```javascript
// Source: derived from existing codex-exec.js pattern + token-log schema
async function runMinimax(prompt, opts) {
  const options = opts || {};
  const model       = options.model      || 'MiniMax-M2.7';
  const maxTokens   = options.maxTokens  || 2000;          // D-04: verbosity control
  const timeoutMs   = options.timeoutMs  || 90000;         // 90s — pre-answer latency can be ~55s
  const temperature = options.temperature || 0.01;          // D-03: must be > 0.0

  // ... API call ...

  // Token shape from OpenAI-compat response.usage:
  // { prompt_tokens, completion_tokens, total_tokens }
  // Map to hook schema: { input_tokens, cached_input_tokens, output_tokens }
  const tokens = {
    input_tokens:        response.usage.prompt_tokens,
    cached_input_tokens: response.usage.prompt_tokens_details?.cached_tokens || 0,
    output_tokens:       response.usage.completion_tokens,
  };

  const { computeCodexCostStrict } = require('./codex-pricing');
  const cost = computeCodexCostStrict(tokens, 'minimax-m2.7') || 0;

  return { success: true, text: response.choices[0].message.content, tokens, cost };
}
```

**Confidence:** HIGH — token field names verified from MiniMax OpenAI-compat API + existing hook schema.

### Pattern 3: `CODEX_PRICING` Extension (D-08)

Add MiniMax to the existing dict. The key name must be lowercase with hyphens to match the pattern used by calling code (`'gpt-5.4'`, `'gpt-5.4-mini'`):

```javascript
// Source: ~/.claude/hooks/codex-pricing.js (lines 28-31, verified 2026-04-03)
// Extend CODEX_PRICING dict:
const CODEX_PRICING = {
  'gpt-5.4':       { input: 2.50, cached_input: 1.25, output: 10.00 },
  'gpt-5.4-mini':  { input: 0.40, cached_input: 0.20, output: 1.60  },
  'minimax-m2.7':  { input: 0.30, cached_input: 0.06, output: 1.20  }, // D-08: per 1M tokens
};
```

**Confidence:** HIGH — pricing verified from synthesis.md section 3.

### Pattern 4: OPUS_PRICING Correction (D-07)

```javascript
// BEFORE (current — wrong, Opus 4.1 pricing):
const OPUS_PRICING = { input: 15.00, cached_input: 3.75, output: 75.00 };

// AFTER (correct — Opus 4.6 pricing):
const OPUS_PRICING = { input: 5.00, cached_input: 1.25, output: 25.00 };
```

**Confidence:** HIGH — confirmed in synthesis.md section 3 footnote: "Multiple reports corrected our earlier research — the $15/$75 pricing is the older Opus 4.1, not Opus 4.6. Opus 4.6 is $5/$25."

### Pattern 5: Settings Block (D-10, D-11)

Add `minimax` block to `.claude/settings.json` alongside the existing `codex` block:

```json
{
  "codex": {
    "routing_enabled": true,
    "fallback_on_error": "prompt_user",
    "attribution_enabled": true,
    "timeout_seconds": 300,
    "preferred_model": "gpt-5.4",
    "api_model": "gpt-5.4-mini"
  },
  "minimax": {
    "enabled": true,
    "model": "MiniMax-M2.7",
    "api_key_env": "MINIMAX_API_KEY",
    "base_url": "https://api.minimax.io/v1",
    "max_tokens_default": 2000,
    "tasks": ["bug-scan", "security-scan", "third-opinion", "context-compress"]
  }
}
```

**Confidence:** HIGH — structure derived from D-10/D-11 decisions; existing `codex` block confirmed from `.claude/settings.json` (verified 2026-04-03).

### Pattern 6: Retry Logic (Claude's Discretion)

Based on the research report's "pragmatic retry policy" and the MiniMax API characteristics:

```javascript
// Recommended: exponential backoff with jitter, max 3 retries for MiniMax
// Rationale: MiniMax has 500 RPM / 20M TPM limits — rate limiting is unlikely
// in single-user terminal workflow; 3 retries covers transient HTTP errors
async function callWithRetry(fn, maxRetries = 3) {
  let lastErr;
  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    try {
      return await fn();
    } catch (err) {
      lastErr = err;
      if (attempt === maxRetries) break;
      // Exponential backoff with jitter: base 1s, max 8s
      const baseDelay = Math.min(1000 * Math.pow(2, attempt), 8000);
      const jitter = Math.random() * 500;
      await new Promise(r => setTimeout(r, baseDelay + jitter));
    }
  }
  throw lastErr;
}
```

**Confidence:** MEDIUM — pattern derived from deep-research-report.md retry guidance; exact backoff values are discretionary.

### Pattern 7: Codex Rate-Limit Detection (D-15)

```javascript
// Source: derived from codex-exec.js parseCodexTokens + D-15 decisions
function isCodexRateLimited(result) {
  if (!result) return false;
  // Check rate_limit_pct from token events
  if (result.rateLimitPct !== null && result.rateLimitPct >= 95) return true;
  // Check exit code + stderr signals
  if (result.error) {
    const msg = result.error.toLowerCase();
    if (msg.includes('rate limit') || msg.includes('quota') || msg.includes('usage limit')) return true;
  }
  // HTTP 429 in JSONL output
  if (result.output && result.output.includes('429')) return true;
  // Timeout with no output
  if (result.error && result.error.includes('timed out') && !result.output) return true;
  return false;
}
```

**Confidence:** HIGH — directly implements D-15.

### Pattern 8: Migration Script Design (D-09)

The migration must be idempotent (D-09 explicit, synthesis specifics section). Key facts for implementation:

- **Token log schema confirmed:** `{ timestamp, session_id, model, source, task_type, ..., tokens: { input, cached_input, output }, cost_usd, rate_limit_pct, project_path, project_name, opus_baseline_usd }` — verified from live `global.jsonl` record.
- **82 records** in `.planning/token-log.jsonl`, **215 records** in `~/.claude/dashboard/global.jsonl`.
- The aggregator (`codex-global-aggregator.js`) uses `computeOpusCost()` from `codex-pricing.js` to compute `opus_baseline_usd`. After fixing `OPUS_PRICING`, re-running the aggregator will recalculate baseline values — but the **per-project token-log.jsonl files** have `opus_baseline_usd` baked in from when records were written.
- The migration script must update `opus_baseline_usd` in each JSONL record and rewrite the file atomically. It then re-runs the aggregator (or rewrites global.jsonl directly).

```javascript
// Migration approach: streaming line-by-line (handles large files without loading all in memory)
// Idempotency: recalculate opus_baseline_usd deterministically from stored tokens field
// Atomic write: write to .tmp file, then fs.renameSync (consistent with existing atomicWriteJSON pattern)
```

**Confidence:** HIGH — schema verified from live data; atomic write pattern confirmed from `codex-global-aggregator.js`.

### Anti-Patterns to Avoid

- **Setting temperature to exactly 0:** MiniMax API enforces `(0.0, 1.0]` — will return an error. Always use `0.01`.
- **Using `response_format: { type: "json_object" }`:** Not supported for M-2.7 (only Text-01 family). Validate JSON client-side if needed.
- **Importing `codex-pricing.js` with relative path `./codex-pricing`:** Works only if the hook's `cwd` is `/home/alucard/.claude/hooks`. Use `require('/home/alucard/.claude/hooks/codex-pricing')` for reliability, or follow the pattern already established in `codex-review-gate.js` (which uses `require('./codex-exec')` — this works because all hooks are in the same directory and spawned from there).
- **Stripping `<think>` content from conversation history:** Not relevant for Phase 8 (single-turn calls only), but important to document for Phases 9-12.
- **Nesting `minimax` block inside `codex` block in settings.json:** Locked by D-10 — must be a sibling key.
- **Using Token Plan key:** Expires with subscription. Pay-as-you-go key (D-12) never expires.

---

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| HTTP retries with backoff | Custom retry loop | AbortController + setTimeout pattern (as in `runGpt54MiniApi`) | Already proven in codebase; handles timeout + retry cleanly |
| Token cost computation | Inline arithmetic | `computeCodexCostStrict(tokens, 'minimax-m2.7')` from `codex-pricing.js` | Single source of truth; strict version returns null for unknown models, making pricing gaps visible |
| Atomic file writes | `fs.writeFileSync` directly | `write-to-temp-then-renameSync` pattern from `codex-global-aggregator.js` | Prevents corruption from concurrent hook executions |
| MiniMax connectivity test | Custom HTTP client | OpenAI SDK with baseURL swap | Zero new dependencies; same SDK already tested in production hooks |

**Key insight:** Every infrastructure piece needed already exists in the hook codebase. Phase 8 is assembly, not invention.

---

## Runtime State Inventory

> Included because Phase 8 changes a pricing constant (`OPUS_PRICING`) that affects values baked into historical records.

| Category | Items Found | Action Required |
|----------|-------------|------------------|
| Stored data | 82 records in `.planning/token-log.jsonl` with `opus_baseline_usd` computed from old pricing ($15/$75). 215 records in `~/.claude/dashboard/global.jsonl` with same issue. | Data migration: recalculate and rewrite `opus_baseline_usd` in all JSONL records using corrected $5/$25 pricing. |
| Live service config | `~/.bashrc` — `MINIMAX_API_KEY` not present (confirmed 2026-04-03). `OPENAI_API_KEY` not verified but assumed set since existing hooks function. | Code + env edit: add `export MINIMAX_API_KEY="..."` to `~/.bashrc`. |
| OS-registered state | No OS-level registration uses "minimax" or "opus" string — hook registration in `~/.claude/settings.json` is file-based, not OS registry. | None — file edit only. |
| Secrets/env vars | `MINIMAX_API_KEY` — does not exist yet. `OPENAI_API_KEY` — used by existing hooks (not renamed). | Create new env var. Existing `OPENAI_API_KEY` unchanged. |
| Build artifacts | None — all hooks are interpreted Node.js scripts with no compiled artifacts or installed packages local to the project. | None. |

**The canonical question:** After `minimax-exec.js` is written and `codex-pricing.js` is patched, what runtime systems still have the old state?
- `~/.bashrc` still has no `MINIMAX_API_KEY` — must be added before connectivity test.
- Historical JSONL files still have stale `opus_baseline_usd` — requires migration script execution.
- `~/.claude/dashboard/dashboard.html` will show wrong savings % until the aggregator re-runs after migration.

---

## Common Pitfalls

### Pitfall 1: MiniMax API Temperature = 0 Rejection
**What goes wrong:** Calling with `temperature: 0` returns a 400 error from the MiniMax API.
**Why it happens:** MiniMax enforces `temperature` must be in `(0.0, 1.0]` — zero is explicitly excluded.
**How to avoid:** Always default to `0.01`. Document in the function signature and JSDoc. Locked by D-03.
**Warning signs:** `400 Bad Request` response containing "temperature" in the error message.

### Pitfall 2: Pre-Answer Latency Up to ~55 Seconds
**What goes wrong:** Hook times out before MiniMax returns a response, even though the request succeeded.
**Why it happens:** MiniMax M-2.7's mandatory `<think>` reasoning phase can take 30-55 seconds on complex prompts before the first output token.
**How to avoid:** Set `timeoutMs: 90000` (90s) minimum — confirmed in synthesis.md section 4 ("Set generous timeouts").
**Warning signs:** Hook exits with timeout error but no 4xx/5xx response code.

### Pitfall 3: Token Field Name Mismatch Between MiniMax and Hook Schema
**What goes wrong:** `tokens.input` is undefined in cost calculation because the OpenAI SDK response uses `prompt_tokens`, not `input_tokens`.
**Why it happens:** The hook token-log schema uses `{ input, cached_input, output }` (short names). The OpenAI SDK's `response.usage` uses `{ prompt_tokens, completion_tokens, total_tokens }`. The `computeOpusCost` function specifically uses short names; `computeCost`/`computeCodexCostStrict` use `input_tokens`.
**How to avoid:** Map field names explicitly in `runMinimax()`. Use `input_tokens: response.usage.prompt_tokens` for the pricing functions; use `input: response.usage.prompt_tokens` for token-log records. Verify the correct key names before writing cost calculation code.
**Warning signs:** `cost_usd: 0` in token-log despite a real API call; `NaN` in savings calculations.

### Pitfall 4: Migration Script Breaks the Aggregator's Dedup Logic
**What goes wrong:** After migration, the aggregator re-reads the updated token-log.jsonl files and creates duplicate records in global.jsonl because the dedup Set uses record content as the hash, not a stable ID.
**Why it happens:** The aggregator uses `timestamp + session_id + model` as the dedup key (inferred from `codex-global-aggregator.js` pattern). If migration changes `opus_baseline_usd` but not the key fields, records should survive dedup. However, if the aggregator hashes the full record content, migrated records will be treated as new.
**How to avoid:** Review the aggregator's dedup logic before writing the migration script. If it deduplicates on content, the migration must also update global.jsonl directly (not rely on re-aggregation). The safer approach: rewrite global.jsonl directly in the migration script using the same atomic write pattern.
**Warning signs:** global.jsonl line count doubles after first SessionStart following migration.

### Pitfall 5: `MINIMAX_API_KEY` Not Available in Hook subprocess Environment
**What goes wrong:** `runMinimax()` gets `undefined` for the API key even though you added it to `~/.bashrc`.
**Why it happens:** Claude Code hooks are spawned as child processes. They inherit the environment from the Claude Code session, not from a new shell. If the session was started before `~/.bashrc` was updated, the new env var is not inherited.
**How to avoid:** After adding `MINIMAX_API_KEY` to `~/.bashrc`, verify connectivity by running a test script directly in terminal (not via Claude Code hook). Then restart the Claude Code session.
**Warning signs:** `runMinimax()` returns `{ success: false, error: "MINIMAX_API_KEY is not set" }`.

### Pitfall 6: `opus_baseline_usd` Migration Makes Future Aggregation Incorrect
**What goes wrong:** After migration rewrites historical records, the next SessionStart aggregation recomputes `opus_baseline_usd` using the new `OPUS_PRICING` from the updated `codex-pricing.js`. If the migration also rewrote those values, you get correct values. But if only the pricing module was updated without migration, future aggregation runs will compute new values for NEW records but old records in global.jsonl still have the wrong baseline — creating a mixed dataset.
**Why it happens:** The aggregator reads from per-project token-log.jsonl and uses `computeOpusCost` to set `opus_baseline_usd`. If the migration does NOT update per-project token-log files (only global.jsonl), the aggregator will recompute from the source files next run and undo the correction.
**How to avoid:** Migration must update BOTH per-project token-log.jsonl files AND global.jsonl. The per-project files are the source of truth; global.jsonl is derived.
**Warning signs:** Dashboard savings % reverts to wrong value after the next session start.

---

## Code Examples

Verified patterns from official sources and existing codebase:

### Basic MiniMax API Call (Non-Streaming)
```javascript
// Source: minimax-m2.7-synthesis.md section 4 + deep-research-report.md lines 70-104
// Adapted for hook module pattern from codex-exec.js runGpt54MiniApi

'use strict';

const MINIMAX_BASE_URL = 'https://api.minimax.io/v1';
const OPENAI_GLOBAL_PATH = '/home/alucard/.npm-global/lib/node_modules/openai';

async function runMinimax(prompt, opts) {
  const options = opts || {};
  const model       = options.model      || 'MiniMax-M2.7';
  const maxTokens   = options.maxTokens  || 2000;     // D-04
  const timeoutMs   = options.timeoutMs  || 90000;    // 90s for ~55s pre-answer latency
  const temperature = options.temperature || 0.01;    // D-03: cannot be exactly 0

  const apiKey = process.env.MINIMAX_API_KEY;
  if (!apiKey) {
    return { success: false, text: '', tokens: null, cost: 0, error: 'MINIMAX_API_KEY is not set' };
  }

  let OpenAI;
  try {
    try { OpenAI = require('openai').OpenAI; }
    catch (e) { OpenAI = require(OPENAI_GLOBAL_PATH).OpenAI; }
  } catch (e) {
    return { success: false, text: '', tokens: null, cost: 0, error: 'openai package not installed: ' + e.message };
  }

  const client = new OpenAI({ baseURL: MINIMAX_BASE_URL, apiKey });
  const controller = new AbortController();
  const timer = setTimeout(() => controller.abort(), timeoutMs);

  try {
    const response = await client.chat.completions.create(
      {
        model,
        messages: [{ role: 'user', content: prompt }],
        max_tokens: maxTokens,
        temperature,
        // NOTE: reasoning_split: true deferred to Phase 10 (adversarial review)
      },
      { signal: controller.signal }
    );

    const usage = response.usage || {};
    const tokens = {
      input_tokens:        usage.prompt_tokens                               || 0,
      cached_input_tokens: (usage.prompt_tokens_details && usage.prompt_tokens_details.cached_tokens) || 0,
      output_tokens:       usage.completion_tokens                           || 0,
    };

    const { computeCodexCostStrict } = require('/home/alucard/.claude/hooks/codex-pricing');
    const cost = computeCodexCostStrict(tokens, 'minimax-m2.7') || 0;

    return {
      success: true,
      text: response.choices[0].message.content,
      tokens,
      cost,
      error: null,
    };
  } catch (e) {
    return { success: false, text: '', tokens: null, cost: 0, error: e.message };
  } finally {
    clearTimeout(timer);
  }
}
```

### Codex Rate-Limit Detection
```javascript
// Source: D-15 decisions + codex-exec.js existing result shape
function isCodexRateLimited(codexResult) {
  if (!codexResult) return false;
  if (codexResult.rateLimitPct !== null && codexResult.rateLimitPct >= 95) return true;
  if (codexResult.error) {
    const msg = codexResult.error.toLowerCase();
    if (msg.includes('rate limit') || msg.includes('quota') || msg.includes('usage limit')) return true;
  }
  if (codexResult.output && codexResult.output.includes('"status":429')) return true;
  if (codexResult.error && codexResult.error.includes('timed out') &&
      (!codexResult.output || codexResult.output.trim() === '')) return true;
  return false;
}
```

### `runWithFallback` Skeleton
```javascript
// Source: D-14 through D-17 decisions
async function runWithFallback(prompt, opts) {
  const { runCodexExec } = require('/home/alucard/.claude/hooks/codex-exec');

  // Step 1: Try Codex CLI
  const codexResult = await runCodexExec(prompt, opts);
  if (codexResult.success) {
    return { success: true, text: codexResult.output, source: 'codex-cli', ...codexResult };
  }

  // Step 2: Check if rate-limited — if so, fall back to MiniMax
  if (isCodexRateLimited(codexResult)) {
    // Log the Codex failure
    logFallbackEvent('codex-rate-limited', codexResult.error, opts);

    const minimaxResult = await runMinimax(prompt, opts);
    if (minimaxResult.success) {
      return { ...minimaxResult, source: 'api-fallback' };
    }

    // Step 3: Both failed — D-17 behavior depends on task type
    // For review tasks (fail-open): return empty pass
    // For execution tasks (fail-closed): prompt user
    return {
      success: false,
      source: 'all-failed',
      codexError: codexResult.error,
      minimaxError: minimaxResult.error,
    };
  }

  // Codex failed for non-rate-limit reason — surface the error directly
  return { success: false, source: 'codex-failed', error: codexResult.error };
}
```

### Atomic JSONL Rewrite (Migration Pattern)
```javascript
// Source: codex-global-aggregator.js atomicWriteJSON pattern (lines 48-52)
const fs = require('fs');

function atomicRewriteJsonl(filePath, transformRecord) {
  const content = fs.readFileSync(filePath, 'utf8');
  const lines = content.split('\n').filter(l => l.trim());
  const updated = lines.map(line => {
    try {
      const record = JSON.parse(line);
      return JSON.stringify(transformRecord(record));
    } catch (e) {
      return line; // preserve unparseable lines unchanged
    }
  });
  const tmp = filePath + '.tmp.' + process.pid;
  fs.writeFileSync(tmp, updated.join('\n') + '\n', 'utf8');
  fs.renameSync(tmp, filePath);
}

// Usage in migration:
const NEW_OPUS_PRICING = { input: 5.00, cached_input: 1.25, output: 25.00 };

function recalculateOpusBaseline(record) {
  if (!record.tokens) return record;
  const inp     = record.tokens.input        || 0;
  const cached  = record.tokens.cached_input || 0;
  const out     = record.tokens.output       || 0;
  record.opus_baseline_usd =
    (inp * NEW_OPUS_PRICING.input +
     cached * NEW_OPUS_PRICING.cached_input +
     out * NEW_OPUS_PRICING.output) / 1_000_000;
  return record;
}
```

---

## Environment Availability

| Dependency | Required By | Available | Version | Fallback |
|------------|------------|-----------|---------|----------|
| Node.js | All hook scripts | Yes | v22.22.0 | — |
| `openai` npm package | `minimax-exec.js` SDK call | Yes (global) | 6.33.0 at `/home/alucard/.npm-global/lib/node_modules/openai` | — |
| `MINIMAX_API_KEY` env var | `runMinimax()` API authentication | **No** — not set in `~/.bashrc` or current env | — | None — must be set before connectivity verification |
| `OPENAI_API_KEY` env var | `runWithFallback()` Codex CLI path | Assumed set (existing hooks functional) | — | — |
| MiniMax API endpoint `api.minimax.io` | Connectivity verification | Unknown — not tested | — | OpenRouter at `openrouter.ai/api/v1` with model `minimax/minimax-m2.7` |
| `codex` CLI | `runWithFallback()` Codex path | Yes | 0.118.0 at `~/.npm-global/bin/codex` | MiniMax direct via `runMinimax()` |
| `.planning/token-log.jsonl` | Migration script | Yes | 82 records | — |
| `~/.claude/dashboard/global.jsonl` | Migration script | Yes | 215 records | — |

**Missing dependencies blocking execution:**
- `MINIMAX_API_KEY` — required for the connectivity verification step. Must be created in MiniMax platform console and added to `~/.bashrc` before Plan 3 can complete.

**Missing dependencies with fallback:**
- MiniMax API connectivity — if `api.minimax.io` is unreachable, OpenRouter (`openrouter.ai/api/v1`, model `minimax/minimax-m2.7`) provides same pricing/interface. Phase 8 uses direct API per D-12; OpenRouter is a deferred option.

---

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Opus 4.1 pricing ($15/$75 per Mtok) | Opus 4.6 pricing ($5/$25 per Mtok) | Opus 4.6 release (2025) | All historical `opus_baseline_usd` values are 3x too high; savings % overreported |
| Two-model routing (Opus + Codex) | Three-model routing (Opus + Codex + MiniMax) | Phase 8 (this phase) | Analysis/review tasks move to MiniMax at 17-21x lower cost than Opus |
| `CODEX_PRICING` covers only gpt-5.4 and gpt-5.4-mini | Extends to include `minimax-m2.7` | Phase 8 | `computeCodexCostStrict` returns null for MiniMax until this phase |

**Deprecated/outdated:**
- `OPUS_PRICING = { input: 15.00, output: 75.00 }`: These are Opus 4.1 prices. Must be replaced with `{ input: 5.00, output: 25.00 }` for Opus 4.6 accuracy.

---

## Open Questions

1. **MiniMax API cached token field name in OpenAI-compat response**
   - What we know: OpenAI SDK returns `response.usage.prompt_tokens_details.cached_tokens` for cache hits.
   - What's unclear: Whether MiniMax populates `prompt_tokens_details` in their OpenAI-compat response, or uses a different field name.
   - Recommendation: In the connectivity verification test (Plan 3), log the full `response.usage` object to determine exact field names. Implement `runMinimax()` with a defensive fallback: `cached_input_tokens: (usage.prompt_tokens_details?.cached_tokens) || (usage.cached_tokens) || 0`.

2. **Whether migration should rewrite source token-log.jsonl files or only global.jsonl**
   - What we know: The aggregator reads per-project token-log.jsonl as source of truth and recomputes `opus_baseline_usd` using `computeOpusCost`. After fixing `OPUS_PRICING`, future aggregation runs will use correct pricing for new records. Old records in per-project files still have stale `opus_baseline_usd` baked in, but the aggregator may or may not re-read those fields (it may recompute from token counts).
   - What's unclear: Whether the aggregator reads `opus_baseline_usd` from JSONL records or always recomputes it from token counts + pricing. Need to check `codex-global-aggregator.js` lines 50-200 before writing migration script.
   - Recommendation: Plan 1 should read the aggregator's merge logic to determine if per-project JSONL migration is strictly necessary, or if fixing `codex-pricing.js` + re-running aggregation is sufficient.

3. **`runWithFallback` fail-open vs fail-closed branching**
   - What we know: D-17 says execution is fail-closed, review is fail-open.
   - What's unclear: How `runWithFallback` knows whether it's being called for execution vs. review — it receives `opts` but the task type may not always be passed.
   - Recommendation: Add `opts.taskCategory` parameter (values: `'execution'` | `'review'`). Default to `'review'` (fail-open) if not specified, since review failures are cheaper than execution blocks.

---

## Sources

### Primary (HIGH confidence)
- `minimax-m2.7-synthesis.md` (local, 2026-04-03) — pricing, API endpoints, gotchas, temperature constraint
- `~/.claude/hooks/codex-exec.js` (local, verified 2026-04-03) — lazy-require pattern, `runGpt54MiniApi` template, return shape
- `~/.claude/hooks/codex-pricing.js` (local, verified 2026-04-03) — current `OPUS_PRICING` values, `CODEX_PRICING` dict pattern
- `~/.claude/hooks/codex-global-aggregator.js` (local, verified 2026-04-03) — atomic write pattern
- `.planning/token-log.jsonl` (local, verified 2026-04-03) — live JSONL schema confirmation (82 records)
- `~/.claude/dashboard/global.jsonl` (local, verified 2026-04-03) — live schema with `opus_baseline_usd` field (215 records)
- `.claude/settings.json` (local, verified 2026-04-03) — existing `codex` settings block structure

### Secondary (MEDIUM confidence)
- `research/deep-research-report.md` (local, 2026-04-03) — retry policy pattern, OpenAI SDK baseURL swap, error handling conventions; sourced from Perplexity deep research
- `08-CONTEXT.md` (local, 2026-04-03) — all implementation decisions D-01 through D-17

### Tertiary (LOW confidence — requires verification)
- MiniMax `prompt_tokens_details.cached_tokens` field availability — assumed by analogy with OpenAI API; not confirmed for MiniMax's OpenAI-compat endpoint. Flag for verification in Plan 3 connectivity test.

---

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH — zero new dependencies; existing package verified at exact path
- Architecture: HIGH — all patterns derived from existing hook source code + locked decisions
- Pricing values: HIGH — confirmed from synthesis document citing multiple research reports
- Pitfalls: HIGH (Pitfalls 1-3, 5) / MEDIUM (Pitfalls 4, 6) — some aggregator internals require reading more source before confirmed
- Migration approach: MEDIUM — depends on Open Question #2 (aggregator merge logic)

**Research date:** 2026-04-03
**Valid until:** 2026-05-03 (30 days — MiniMax pricing/API is stable; OpenAI SDK changes infrequently)
