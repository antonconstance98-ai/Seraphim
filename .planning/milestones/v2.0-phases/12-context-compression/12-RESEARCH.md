# Phase 12: Context Compression - Research

**Researched:** 2026-04-03
**Domain:** Node.js hook script, MiniMax M-2.7 summarization, Claude Code PostToolUse / context monitor integration
**Confidence:** HIGH

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions

**D-01:** Threshold-based auto-compression plus manual utility. Three auto-triggers:
1. `gsd-context-monitor.js` detects >60% context usage → compress earlier conversation context
2. A tool output exceeds 10K tokens → compress before injecting as `additionalContext`
3. A git diff exceeds 8K characters → compress before sending to review hooks

**D-02:** Also exposed as `require('./minimax-compress')` for any hook to call manually. Function signature: `compress(text, { maxOutputTokens, purpose })` where purpose guides the compression style ("summarize code diff", "condense tool output", etc.).

**D-03:** All thresholds configurable in the `minimax` settings block: `"compress_context_pct": 60`, `"compress_tool_output_tokens": 10000`, `"compress_diff_chars": 8000`.

**D-04:** `PostToolUse` — wrap large tool outputs before they consume context. Check `tool_result` length against threshold. If over, compress and output the summary as `additionalContext` instead.

**D-05:** `gsd-context-monitor.js` integration — when context usage hits the warning threshold, the monitor calls `minimax-compress` to summarize accumulated context and injects the compressed version.

**D-06:** Utility callable from review hooks — `codex-review-gate.js` can compress large diffs before sending to Codex/MiniMax review, keeping review costs down.

**D-07:** Compression prompt includes the `purpose` parameter so MiniMax knows what to preserve. For code diffs: preserve file names, function signatures, and the nature of changes. For tool outputs: preserve key data points, errors, and actionable information. For conversations: preserve decisions, blockers, and action items.

**D-08:** Output always starts with `[Compressed from ~{N}K tokens]` header so downstream consumers know they're reading a summary, not the original.

**Hook integration points:** `PostToolUse` and the context monitor. Also a `require()` utility.

### Claude's Discretion

- Exact compression prompts for each purpose type
- Whether to cache compressed results (avoid re-compressing the same content)
- Compression ratio targets (how aggressively to summarize)
- Whether to store originals alongside compressed versions for audit

### Deferred Ideas (OUT OF SCOPE)

- Streaming compression (compress as content arrives rather than after) — complex, evaluate if batch compression is too slow
- Compression quality metrics (how much information is lost) — hard to measure, defer to manual spot-checks
</user_constraints>

---

## Summary

Phase 12 creates `minimax-compress.js` — a dual-mode module that (1) acts as a standalone PostToolUse hook filtering large tool outputs, and (2) exports a `compress(text, opts)` function that any other hook can `require()` to compress large inputs before sending them onward. The module is also integrated into `gsd-context-monitor.js` to trigger compression when session context exceeds the 60% usage threshold.

The core implementation is straightforward: measure input size using a character-based token approximation (4 chars ≈ 1 token, industry standard for English/code), compare against configurable thresholds, build a purpose-aware prompt, call `runMinimax()` from the existing `minimax-exec.js` module, and return the compressed text prefixed with `[Compressed from ~{N}K tokens]`. All three thresholds are already in the project settings spec (D-03). The `additionalContext` injection limit (10,000 chars) is a hard ceiling the planner must enforce in output sizing.

The primary technical risk is the `gsd-context-monitor.js` integration: the monitor currently fires on every PostToolUse call and is kept synchronous for minimal overhead. Adding an async MiniMax call inside it requires care — the monitor must call compression only when the threshold is actually hit, must not hang the hook chain, and must fail silently if MiniMax is unavailable. The correct pattern (verified from Phase 11's minimax-post-scan.js) is lazy-require of `minimax-compress`, run the async compression, and emit `additionalContext`. Since PostToolUse hooks are already async-capable in this codebase, this is safe.

**Primary recommendation:** Implement `minimax-compress.js` as a Node.js module following the exact minimax-post-scan.js structural pattern (stdin/stdout JSON, lazy-require minimax-exec, fail-silent). The context-monitor integration should `lazy-require` minimax-compress only when threshold is hit, making the hot path (no compression needed) zero-overhead.

---

## Standard Stack

### Core

| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| Node.js built-ins (`fs`, `path`, `os`) | v22.22.0 (installed) | File I/O, path resolution, tmpdir | Same pattern as all 11 prior hooks in this project |
| `minimax-exec.js` | Phase 8 (local) | MiniMax API call via `runMinimax()` | The established SDK wrapper — all MiniMax calls go through it |
| `codex-pricing.js` | Phase 5 (local) | `computeCodexCostStrict()` for token logging | Existing cost accounting pattern |

### Supporting

| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| `openai` npm package | 6.33.0 (installed globally at `~/.npm-global`) | Indirect — loaded inside minimax-exec.js | Never required directly by minimax-compress.js; minimax-exec.js handles the SDK |

### Alternatives Considered

| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| 4-char-per-token heuristic | `tiktoken` npm package | tiktoken gives exact token counts but requires a 1.5MB WASM binary install; the 4-char heuristic is ±15% accurate, sufficient for threshold comparisons (D-03 thresholds are set conservatively) |
| Lazy-require minimax-exec | Top-level require | Top-level require pays the module load cost on every PostToolUse call, even when no compression is needed; lazy-require keeps the hot path fast (same decision as Phase 11) |
| Separate PostToolUse hook file | Combine into existing gsd-context-monitor.js | Separate file preserves single-responsibility and makes it independently testable; consistent with Phase 11 pattern |

**Installation:** No new packages required. All dependencies are already present.

---

## Architecture Patterns

### Recommended Project Structure

```
~/.claude/hooks/
├── minimax-compress.js      # NEW: Phase 12 — dual-mode compression module
├── gsd-context-monitor.js   # MODIFIED: Phase 12 — add compress call at threshold
├── minimax-exec.js          # Existing: runMinimax() — no changes needed
├── codex-pricing.js         # Existing: computeCodexCostStrict() — no changes needed
└── ...                      # All other hooks unchanged

.claude/settings.json        # MODIFIED: add compress_context_pct, compress_tool_output_tokens,
                             # compress_diff_chars to minimax block (D-03)
```

### Pattern 1: Dual-Mode Module (Hook + require() Library)

**What:** `minimax-compress.js` operates in two modes determined by whether stdin contains valid hook JSON. When loaded as a PostToolUse hook it reads stdin and filters large tool outputs. When `require()`d by other scripts, it exports `compress(text, opts)` directly.

**When to use:** Any time a single file needs to serve as both a runnable hook script and a reusable library. This pattern is not currently used in the codebase — `minimax-exec.js` is library-only (no stdin handling). Phase 12 introduces the first dual-mode module.

**Implementation approach:** Check `require.main === module` to distinguish hook execution from library import:

```javascript
// Source: established Node.js pattern; verified against Phase 11 minimax-post-scan.js structure
'use strict';

const fs = require('fs');
const path = require('path');
const os = require('os');

// Exported library function — callable from any hook via require()
async function compress(text, opts) {
  const options = opts || {};
  const purpose = options.purpose || 'condense content';
  const maxOutputTokens = options.maxOutputTokens || 1500;
  const inputChars = text.length;
  const approxInputTokens = Math.round(inputChars / 4);

  const { runMinimax } = require('/home/alucard/.claude/hooks/minimax-exec');

  const prompt = buildCompressionPrompt(text, purpose);
  const result = await runMinimax(prompt, {
    maxTokens: maxOutputTokens,
    timeoutMs: 60000,  // compression can take longer than bug scan
    systemPrompt: 'You are a technical summarizer. Preserve all actionable information.'
  });

  if (!result.success) {
    return { success: false, text: '', error: result.error };
  }

  // Strip <think> reasoning block — downstream consumers only need the summary
  const compressed = result.text.replace(/<think>[\s\S]*?<\/think>\s*/g, '').trim();
  const header = `[Compressed from ~${Math.round(approxInputTokens / 1000)}K tokens]\n`;

  return {
    success: true,
    text: header + compressed,
    tokens: result.tokens,
    cost: result.cost,
    originalChars: inputChars,
    compressedChars: (header + compressed).length
  };
}

// Hook execution — only when run directly as a PostToolUse hook
if (require.main === module) {
  runAsHook();
}

module.exports = { compress };
```

### Pattern 2: Character-Based Token Estimation

**What:** Approximate token count as `Math.round(chars / 4)`. Used for threshold comparison only — not for billing.

**When to use:** Whenever exact token counting is unavailable and you need a fast estimate to decide whether to compress. Accurate to ±15% for English/code text.

**Why 4 chars per token:** The GPT tokenizer averages ~4 characters per token for English text. This approximation is consistent with the existing `MAX_DIFF_CHARS` pattern in `minimax-post-scan.js` (3,000 chars = ~750 tokens).

```javascript
// Token estimation — compare against D-03 thresholds
const CHARS_PER_TOKEN_APPROX = 4;

function estimateTokens(text) {
  return Math.round(text.length / CHARS_PER_TOKEN_APPROX);
}

function estimateCharsFromTokens(tokens) {
  return tokens * CHARS_PER_TOKEN_APPROX;
}

// D-03 threshold check (D-03: compress_tool_output_tokens = 10000)
// 10,000 tokens * 4 = 40,000 chars
const toolOutputThresholdChars = (config.compress_tool_output_tokens || 10000) * CHARS_PER_TOKEN_APPROX;
```

### Pattern 3: Purpose-Aware Compression Prompts (D-07)

**What:** Different `purpose` values produce different compression instructions to MiniMax.

**When to use:** Always — purpose drives what information is preserved. A code diff summary must keep file paths and function names. A conversation summary must keep decisions and action items.

```javascript
// Source: derived from D-07 decisions and minimax-m2.7-synthesis.md §6
function buildCompressionPrompt(text, purpose) {
  const PRESERVE_RULES = {
    'summarize code diff': [
      'File names and paths',
      'Function and method signatures added or removed',
      'The nature of each change (bug fix, feature, refactor)',
      'Error handling additions or removals',
    ],
    'condense tool output': [
      'Error messages and stack traces (verbatim)',
      'Key data values and counts',
      'Actionable warnings or failures',
      'Success/failure status',
    ],
    'compress conversation': [
      'Decisions made and their rationale',
      'Blockers and open questions',
      'Action items and next steps',
      'File paths and function names mentioned',
    ],
    'condense file content': [
      'Function and class signatures',
      'Public API surface',
      'Important constants and configuration',
      'Critical logic flows',
    ],
  };

  const rules = PRESERVE_RULES[purpose] || ['All actionable and technical information'];
  const ruleList = rules.map(r => `- ${r}`).join('\n');

  return `Summarize the following content as concisely as possible while preserving:\n${ruleList}\n\nDo not add commentary. Output only the compressed summary.\n\n---\n${text}`;
}
```

### Pattern 4: gsd-context-monitor.js Extension (D-05)

**What:** Add compression call inside the existing WARNING threshold branch of `gsd-context-monitor.js`. Only fires when `used_pct >= compress_context_pct` (default 60%).

**When to use:** When the context monitor detects >60% usage.

**Key constraint:** The monitor's existing `additionalContext` output is a warning string. The extension must decide: emit the compression summary instead of or in addition to the warning. Recommendation: emit both — warning first, then compressed summary if compression succeeds. This gives the agent the warning plus an immediate condensed view.

```javascript
// Inside gsd-context-monitor.js — after threshold detection, before building message
// Only attempt compression if context is at warning level AND config enables it
const compressThreshold = (minimaxConfig && typeof minimaxConfig.compress_context_pct === 'number')
  ? minimaxConfig.compress_context_pct
  : 60;  // Default: compress at 60% usage

if (usedPct >= compressThreshold && minimaxConfig && minimaxConfig.enabled !== false) {
  try {
    // Lazy-require to keep the hot path (no compression needed) zero-overhead
    const { compress } = require('/home/alucard/.claude/hooks/minimax-compress');
    // NOTE: gsd-context-monitor.js currently uses synchronous code.
    // The on('end') handler must become async to await compress().
    // See "Common Pitfalls" section for the async migration pattern.
    const compressionResult = await compress(contextText, {
      purpose: 'compress conversation',
      maxOutputTokens: 1200
    });
    if (compressionResult.success) {
      message += '\n\n' + compressionResult.text;
    }
  } catch (_) {
    // Fail-silent — compression is advisory, never blocks the warning
  }
}
```

### Pattern 5: PostToolUse Hook — Tool Output Compression (D-04)

**What:** In PostToolUse mode, read `data.tool_result`, check if length exceeds `compress_tool_output_tokens` threshold (in chars), compress if so, and output as `additionalContext`.

**Key constraint:** The `tool_result` field in PostToolUse stdin is a string (the raw output of the tool). Length is measured in characters; threshold conversion uses 4 chars/token.

```javascript
// PostToolUse stdin shape (verified from codex-token-logger.js and minimax-post-scan.js):
// {
//   session_id: string,
//   cwd: string,
//   tool_name: string,          // "Bash", "Read", "Write", etc.
//   tool_input: object,         // parameters passed to the tool
//   tool_result: string,        // raw output from the tool
// }

const toolResult = data.tool_result || '';
const toolOutputThresholdChars = (minimaxConfig.compress_tool_output_tokens || 10000) * 4;

if (toolResult.length > toolOutputThresholdChars) {
  const result = await compress(toolResult, {
    purpose: 'condense tool output',
    maxOutputTokens: 1500
  });
  if (result.success) {
    process.stdout.write(JSON.stringify({
      hookSpecificOutput: {
        hookEventName: 'PostToolUse',
        additionalContext: result.text
      }
    }));
    process.exit(0);
  }
}
```

### Anti-Patterns to Avoid

- **Top-level require of minimax-exec:** Every PostToolUse call loads the module even when no compression is needed. Use lazy-require inside the conditional — only pay SDK load cost when compression actually fires. (Phase 11 established this pattern: "Lazy-require minimax-exec and codex-pricing only after code-file and diff checks pass.")
- **Blocking on compression failure:** If MiniMax returns an error, the hook must exit(0) silently and let the original tool result pass through. Never block tool execution.
- **outputting compressed text longer than 10,000 chars:** `additionalContext` has a hard 10,000 char limit. The `maxOutputTokens` passed to `runMinimax` must keep output within this bound. At 4 chars/token, 2,000 output tokens ≈ 8,000 chars — safe ceiling.
- **Making gsd-context-monitor.js async without proper stdin handling:** The monitor's `on('end')` callback must become `async` to use `await compress()`. Forgetting to add `async` causes silent failures (Promises returned but not awaited, always resolves before compression completes).
- **Re-compressing already-compressed content:** If the `[Compressed from ~{N}K tokens]` header is detected in `tool_result`, skip compression. Prevents double-compression of content that was already compressed by an earlier hook run.

---

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| MiniMax API call | Custom OpenAI SDK instantiation | `runMinimax()` from `minimax-exec.js` | Already has retry logic, AbortController timeout, token parsing, cost computation, cached token field fallback, and `<think>` tag handling |
| Token counting for billing | Custom tokenizer | `computeCodexCostStrict()` from `codex-pricing.js` | Already handles the minimax-m2.7 pricing entry correctly |
| Threshold configuration | Hardcoded constants | Read from `.claude/settings.json` minimax block | D-03 requires configurability; settings.json already has the `minimax` block |
| Think-tag stripping | Custom regex | Copy the exact regex from minimax-post-scan.js: `result.text.replace(/<think>[\s\S]*?<\/think>\s*/g, '').trim()` | Already tested and handles multiline reasoning blocks |

**Key insight:** The compression utility is a thin wrapper around `runMinimax()`. The bulk of the complexity (API retries, timeouts, SDK loading, token logging schema) is already solved. Phase 12 only needs to add: size measurement, threshold comparison, prompt construction, and output formatting.

---

## Common Pitfalls

### Pitfall 1: gsd-context-monitor.js Sync/Async Mismatch

**What goes wrong:** `gsd-context-monitor.js` currently uses a synchronous `process.stdin.on('end', () => { ... })` callback. If the compress call is added with `await` inside a non-async function, the await is silently ignored and compression never fires.

**Why it happens:** JavaScript's `await` keyword only works inside `async` functions. Dropping it into a regular callback has no error — it just doesn't work.

**How to avoid:** Change `on('end', () => {` to `on('end', async () => {`. Then `await compress(...)` works as expected.

**Warning signs:** Compression log entries never appear in token-log.jsonl even when context exceeds threshold.

### Pitfall 2: tool_result May Not Be a String

**What goes wrong:** `data.tool_result` in PostToolUse is usually a string, but may be `null`, `undefined`, or in edge cases an object (e.g., Bash commands that produce no output). Calling `.length` on null throws.

**Why it happens:** Claude Code's hook protocol sends `tool_result` as the raw tool output. Some tools (like file writes) may return empty strings or null.

**How to avoid:** Always guard: `const toolResult = typeof data.tool_result === 'string' ? data.tool_result : '';` before measuring length.

**Warning signs:** `TypeError: Cannot read properties of null (reading 'length')` in hook stderr.

### Pitfall 3: additionalContext Exceeds 10,000 Char Limit

**What goes wrong:** The `[Compressed from ~{N}K tokens]` header plus the MiniMax output exceeds Claude Code's 10,000 char cap for `additionalContext`. The text gets silently truncated or rejected.

**Why it happens:** MiniMax's verbosity tax (~4x output tokens vs other models) can produce longer-than-expected summaries even with `maxTokens: 1500`.

**How to avoid:** After receiving the compressed text, truncate if necessary before writing to `additionalContext`:
```javascript
const MAX_CONTEXT_CHARS = 9500; // leave 500 char buffer
const output = result.text.slice(0, MAX_CONTEXT_CHARS);
```

**Warning signs:** Opus behaves as if it never received the compression summary even though the hook ran successfully.

### Pitfall 4: Double-Compression Loop

**What goes wrong:** A large tool output gets compressed and injected as `additionalContext`. On the next PostToolUse call, the compressed text (now in `tool_result` of a subsequent read) triggers compression again.

**Why it happens:** The `[Compressed from ~{N}K tokens]` header appears in the output, but if the hook doesn't check for it, it treats the compressed text as a new large input.

**How to avoid:** Add a guard at the start of PostToolUse mode:
```javascript
if (toolResult.startsWith('[Compressed from ~')) {
  process.exit(0); // Already compressed — skip
}
```

**Warning signs:** Token log shows multiple compression calls for the same session within seconds.

### Pitfall 5: MiniMax Timeout on Large Context

**What goes wrong:** Compressing a long conversation context (several thousand tokens) takes longer than the hook timeout, causing the hook to exit with no output.

**Why it happens:** MiniMax has pre-answer latency up to ~55s on first calls. The PostToolUse hook timeout in settings.json is the binding constraint.

**How to avoid:** Register `minimax-compress.js` with a generous timeout (60-90s) in settings.json. Use `timeoutMs: 60000` inside the runMinimax call as the AbortController fence. Keep compression inputs capped (e.g., only compress up to 50K chars of context, not the entire history).

**Warning signs:** Hook exits with timeout logged but no `additionalContext` in Claude's view.

---

## Code Examples

### Verified Pattern: Token Logging (from minimax-post-scan.js)

```javascript
// Source: /home/alucard/.claude/hooks/minimax-post-scan.js lines 311-329
// This is the exact logging schema used by all Phase 11 hooks — Phase 12 must match it
if (result.tokens) {
  try {
    const { computeCodexCostStrict } = require('/home/alucard/.claude/hooks/codex-pricing');
    const record = {
      timestamp:  new Date().toISOString(),
      session_id: data.session_id || null,
      model:      'minimax-m2.7',
      source:     'api',
      task_type:  'context-compress',  // <-- Phase 12 task_type
      tokens: {
        input:        result.tokens.input_tokens        || 0,
        cached_input: result.tokens.cached_input_tokens || 0,
        output:       result.tokens.output_tokens       || 0
      },
      cost_usd: computeCodexCostStrict(result.tokens, 'minimax-m2.7') || 0
    };
    const logPath = path.join(cwd, '.planning', 'token-log.jsonl');
    fs.mkdirSync(path.dirname(logPath), { recursive: true });
    fs.appendFileSync(logPath, JSON.stringify(record) + '\n', 'utf8');
  } catch (_logErr) {
    // Token logging errors are non-fatal
  }
}
```

### Verified Pattern: Reading Project minimax Config (from minimax-post-scan.js)

```javascript
// Source: /home/alucard/.claude/hooks/minimax-post-scan.js lines 261-276
const cwd = data.cwd || process.cwd();
let minimaxConfig = null;
try {
  const projectSettingsPath = path.join(cwd, '.claude', 'settings.json');
  const rawSettings = fs.readFileSync(projectSettingsPath, 'utf8');
  const projectSettings = JSON.parse(rawSettings);
  minimaxConfig = projectSettings.minimax || null;
} catch (_configErr) {
  // No project settings -- use defaults
}

if (minimaxConfig && minimaxConfig.enabled === false) {
  process.exit(0);
}
```

### Verified Pattern: Think-Tag Stripping (from minimax-post-scan.js)

```javascript
// Source: /home/alucard/.claude/hooks/minimax-post-scan.js lines 340-344
const findingsClean = findings.replace(/<think>[\s\S]*?<\/think>\s*/g, '').trim();
```

### Settings.json minimax Block (D-03 additions needed)

The current project `.claude/settings.json` minimax block:
```json
{
  "minimax": {
    "enabled": true,
    "model": "MiniMax-M2.7",
    "api_key_env": "MINIMAX_API_KEY",
    "base_url": "https://api.minimax.io/v1",
    "max_tokens_default": 2000,
    "scan_skip_threshold": 5,
    "tasks": ["bug-scan", "security-scan", "third-opinion", "context-compress"]
  }
}
```

Phase 12 must add three new fields (D-03):
```json
{
  "minimax": {
    "enabled": true,
    "model": "MiniMax-M2.7",
    "api_key_env": "MINIMAX_API_KEY",
    "base_url": "https://api.minimax.io/v1",
    "max_tokens_default": 2000,
    "scan_skip_threshold": 5,
    "compress_context_pct": 60,
    "compress_tool_output_tokens": 10000,
    "compress_diff_chars": 8000,
    "tasks": ["bug-scan", "security-scan", "third-opinion", "context-compress"]
  }
}
```

### Global Settings Hook Registration

The global `~/.claude/settings.json` PostToolUse chain currently ends with `minimax-post-scan.js` (timeout 30s). Phase 12 adds `minimax-compress.js` after it:

```json
{
  "type": "command",
  "command": "node \"/home/alucard/.claude/hooks/minimax-compress.js\"",
  "timeout": 90
}
```

90-second timeout accounts for MiniMax pre-answer latency (~55s) plus network overhead. The hook self-exits immediately (process.exit(0)) when no compression is needed, so the timeout is only paid on actual compression calls.

---

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Truncate large inputs arbitrarily | Compress with a cheap fast model before sending to expensive model | Phase 12 (this phase) | Preserves information while reducing Opus token spend |
| Single-mode scripts (either hook OR library) | Dual-mode via `require.main === module` | Phase 12 (this phase) | First time this pattern is used in this codebase |
| Fixed hardcoded thresholds | Configurable thresholds in settings.json minimax block | Phase 12 (this phase) | Consistent with D-03 decision |

**Relevant baseline from synthesis doc (§6):**
- 500K token codebase → MiniMax compress → 50K summary: total cost $1.71
- vs. Opus directly on 500K: $17.50 (10x more expensive)

---

## Open Questions

1. **What is the "context text" to compress in gsd-context-monitor.js?**
   - What we know: The monitor reads metrics from `/tmp/claude-ctx-{session_id}.json`, which contains `remaining_percentage` and `used_pct`. It does NOT have direct access to the conversation history.
   - What's unclear: The bridge file contains usage metrics, not the actual conversation text. To compress "earlier conversation context", Phase 12 either needs to (a) compress whatever large tool output just triggered the threshold, or (b) have access to conversation text from another source. The CONTEXT.md says "compress earlier conversation context" but the current monitor only has usage percentages.
   - Recommendation: For MVP, treat the context compression in the monitor as injecting a note to Opus suggesting it should summarize what it knows so far (a prompt-based approach, not a literal text compression). Alternatively, limit D-05 to compressing the current hook chain's `additionalContext` payload rather than the full conversation. The planner should clarify scope with a decision comment.

2. **Should minimax-compress.js register as a separate PostToolUse hook, or extend gsd-context-monitor.js?**
   - What we know: Phase 11 added `minimax-post-scan.js` as a separate hook (4th in the PostToolUse chain). The CONTEXT.md says "Hook integration via PostToolUse".
   - What's unclear: Whether tool output compression should be a 5th hook in the chain (separate file) or incorporated into gsd-context-monitor.js (which already fires PostToolUse).
   - Recommendation: Separate hook file for tool output compression (consistent with Phase 11 pattern). The context-monitor integration is a modification to gsd-context-monitor.js only. This preserves single-responsibility.

3. **MINIMAX_API_KEY is not set in the shell environment at research time.**
   - What we know: `minimax-exec.js` returns `{ success: false, error: 'MINIMAX_API_KEY is not set' }` gracefully. All hooks are fail-silent.
   - What's unclear: Whether the key is set in the Claude Code process environment (where hooks run) vs the terminal shell. Phase 8 verified connectivity, so the key must be available to the hook process somehow.
   - Recommendation: Plan must include a verification step: run `node -e "console.log(process.env.MINIMAX_API_KEY ? 'SET' : 'NOT SET')"` inside a Claude Code Bash call (not in terminal) to confirm hook-level access.

---

## Environment Availability

| Dependency | Required By | Available | Version | Fallback |
|------------|------------|-----------|---------|----------|
| Node.js | All hook scripts | Yes | v22.22.0 | — |
| `openai` npm package | `minimax-exec.js` (indirect) | Yes | 6.33.0 at `~/.npm-global` | — |
| `minimax-exec.js` | `compress()` function | Yes | Phase 8 complete | — |
| `codex-pricing.js` | Token logging | Yes | Phase 5 complete | — |
| MiniMax API (`api.minimax.io`) | All compressions | Assumed yes (Phase 8 verified) | MiniMax-M2.7 | Fail-silent: skip compression, pass original |
| MINIMAX_API_KEY env var | Hook process | Unknown in terminal; assumed yes in hook process | — | Fail-silent: compression skipped, returns success:false |

**Missing dependencies with no fallback:** None. All compression calls are advisory and fail-silent.

**Missing dependencies with fallback:** MINIMAX_API_KEY missing → `runMinimax()` returns `success: false` → compression skipped, original content passed through unchanged.

---

## Validation Architecture

> Skipped: `workflow.nyquist_validation` is explicitly `false` in `.planning/config.json`.

---

## Sources

### Primary (HIGH confidence)

- `/home/alucard/.claude/hooks/minimax-exec.js` — `runMinimax()` interface, verified function signature and return shape
- `/home/alucard/.claude/hooks/minimax-post-scan.js` — Structural pattern for dual-responsibility hook (stdin handling + advisory output + token logging)
- `/home/alucard/.claude/hooks/gsd-context-monitor.js` — Full implementation read; confirmed where compression call must be inserted
- `/home/alucard/projects/Claude_X_Codex/.claude/settings.json` — Current minimax config block structure
- `/home/alucard/.claude/settings.json` — Current hook registration in PostToolUse chain (4 hooks including minimax-post-scan at timeout:30)
- `/home/alucard/projects/Claude_X_Codex/minimax-m2.7-synthesis.md` §6 — Two-stage context compression cost analysis ($1.71 vs $17.50)
- `.planning/phases/12-context-compression/12-CONTEXT.md` — All locked decisions D-01 through D-08

### Secondary (MEDIUM confidence)

- `/home/alucard/.claude/hooks/codex-token-logger.js` — Confirmed `tool_result` field name and string type in PostToolUse stdin payload
- `.planning/STATE.md` — Phase 11 decisions: "Lazy-require minimax-exec and codex-pricing only after code-file and diff checks pass" (directly applicable to Phase 12)

### Tertiary (LOW confidence)

- None required — all findings verified against existing code in the repository.

---

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH — all dependencies verified as installed and working from Phases 8-11
- Architecture patterns: HIGH — patterns derived directly from Phase 11 code that was just written and verified
- Pitfalls: HIGH — derived from actual implementation decisions in STATE.md and direct code inspection; not speculation
- Open Questions: MEDIUM — the context text availability question requires a planner decision

**Research date:** 2026-04-03
**Valid until:** 2026-05-03 (stable stack; all dependencies are local modules under version control)
