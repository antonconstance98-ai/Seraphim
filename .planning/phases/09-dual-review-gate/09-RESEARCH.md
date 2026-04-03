# Phase 9: Dual Review Gate - Research

**Researched:** 2026-04-03
**Domain:** Node.js hook modification — parallel async execution, verdict merge, token logging
**Confidence:** HIGH

---

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions

- **D-01:** Either model blocks — if EITHER Codex or MiniMax flags an issue, the response is BLOCKED. Most conservative approach.
- **D-02:** Both verdicts reported separately in the block reason. Format: "Codex found: [issue]. MiniMax found: [issue]." or "Codex: PASS. MiniMax found: [issue]."
- **D-03:** Use `Promise.all` to run Codex CLI (`runCodexExec`) and MiniMax API (`runMinimax`) simultaneously.
- **D-04:** If Codex is rate-limited, MiniMax review still runs independently (graceful degradation via `runWithFallback` from Phase 8). The review becomes single-model, not zero-model.
- **D-05:** If MiniMax fails (API error, timeout), Codex review still proceeds. MiniMax failure is fail-open — existing Codex-only behavior is the fallback.
- **D-06:** Log both models' reviews as separate entries in `token-log.jsonl`. Fields: `model: 'gpt-5.4'` and `model: 'minimax-m2.7'`, both with `task_type: 'review'`, `review_task_type: [feature|security|test-gen|bulk-ops]`.
- **D-07:** Track `dual_review: true` flag in log entries so cost reporting can show dual vs single review costs.

### Claude's Discretion

- Timeout values for MiniMax review call (suggest matching Codex's 120s)
- How to truncate diff for MiniMax if it exceeds context limits (MiniMax has 205K vs Codex's larger window)
- Review prompt adaptation for MiniMax (same prompt as Codex or tailored)

### Deferred Ideas (OUT OF SCOPE)

None — discussion stayed within phase scope.

</user_constraints>

---

## Summary

Phase 9 modifies `codex-review-gate.js` (the Stop hook) to run Codex and MiniMax reviews in parallel using `Promise.all`. The Phase 8 foundation is fully in place: `minimax-exec.js` exports `runMinimax()` and `runWithFallback()`, `codex-pricing.js` has MiniMax pricing, and the project `settings.json` has the `minimax` config block. The hook file itself is the only thing that changes.

The change is surgical: the single `await runCodexExec(...)` call is replaced with a `Promise.all([runCodexExec(...), runMinimax(...)])` pattern. Verdict merge, per-model token logging, and output formatting then happen over two results instead of one. Everything else in the hook — diff collection, task classification, loop guard, fail-open error handling — stays exactly as it is.

The two non-trivial design decisions left to Claude's discretion are: (1) timeout for the MiniMax call (120s matches Codex, which is safe given documented MiniMax pre-answer latency up to 55s), and (2) whether to differentiate review prompts between the two models to maximize the "different perspective" value.

**Primary recommendation:** Make the MiniMax prompt security/edge-case focused while Codex keeps the existing correctness/architecture focus. This adds genuine diversity of perspective, directly realizing the project's core value proposition. Diff truncation for MiniMax should match the existing 8000-char limit — MiniMax's 205K context is not the constraint; uniformity and predictability are.

---

## Standard Stack

### Core (already installed — no new packages)

| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| Node.js built-in `Promise.all` | v22.22.0 | Parallel async execution | Zero dependency; correct tool for concurrent async tasks |
| `minimax-exec.js` `runMinimax()` | Phase 8 output | MiniMax API call | Already tested and deployed; this phase is a consumer, not a builder |
| `codex-exec.js` `runCodexExec()` | Existing | Codex CLI invocation | Unchanged from current hook |
| `codex-pricing.js` `computeCost` / `computeCodexCostStrict` | Existing | Per-model cost computation | Both models already have pricing entries |

### Supporting

| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| `minimax-exec.js` `runWithFallback()` | Phase 8 output | Rate-limit-aware Codex call | Use for the Codex leg so D-04 (graceful degradation) is automatic |

**Installation:** No new packages required. All dependencies are in place from prior phases.

---

## Architecture Patterns

### How the Current Hook Invokes Codex (to be modified)

```javascript
// CURRENT: single sequential call
const result = await runCodexExec(reviewPrompt, {
  cwd,
  timeoutMs: 120000,
  model: 'gpt-5.4'
});
```

### Pattern: Parallel Execution via Promise.all

The core change replaces the single Codex call with parallel execution. Both promises run concurrently; `Promise.all` resolves when BOTH complete (or when the first rejects if using plain `.all`).

**Important:** Use `Promise.allSettled` rather than `Promise.all` if either call might throw. However, both `runCodexExec` and `runMinimax` are designed to catch internally and return `{ success: false, ... }` rather than throw, so `Promise.all` is safe. The CONTEXT.md specifies `Promise.all` (D-03) — this is consistent with the module contracts.

```javascript
// Source: Node.js built-in Promise.all, D-03 from CONTEXT.md
const [codexResult, minimaxResult] = await Promise.all([
  runWithFallback(codexReviewPrompt, {
    cwd,
    timeoutMs: 120000,
    model: 'gpt-5.4',
    taskCategory: 'review'      // fail-open if both models fail
  }),
  runMinimax(minimaxReviewPrompt, {
    maxTokens: 2000,
    timeoutMs: 120000           // Claude's discretion: match Codex timeout
  })
]);
```

**Why `runWithFallback` for the Codex leg:** `runWithFallback` already implements D-04 (graceful degradation on rate-limit). When Codex is rate-limited, `runWithFallback` auto-escalates to MiniMax, which means MiniMax handles both legs. For Phase 9, however, D-03 says the two calls are independent, and D-04 says "MiniMax review still runs independently." Using `runWithFallback` for Codex + direct `runMinimax` for MiniMax is the cleanest separation — each model's result is tracked independently.

**Alternative:** Call `runCodexExec` directly (bypassing fallback chain) so the two legs are truly independent. This is simpler and avoids double-MiniMax spending if Codex is rate-limited. Recommendation: use direct `runCodexExec` for the Codex leg and direct `runMinimax` for the MiniMax leg, consistent with D-03 intent.

### Pattern: Fail-Open Per Model

D-05: MiniMax failure is fail-open — treat it as if it never ran, fall through to Codex-only verdict.
D-04: Codex rate-limit causes graceful degradation to MiniMax-only verdict.

```javascript
// After Promise.all resolves:
const codexOk   = codexResult.success;
const minimaxOk = minimaxResult.success;

// At least one model must have succeeded for a meaningful review
if (!codexOk && !minimaxOk) {
  // Both failed — fail-open, exit 0
  process.exit(0);
}
```

### Pattern: Verdict Merge (D-01, D-02)

```javascript
// Reuse existing parseVerdict() unchanged — it operates on a string
const codexVerdict   = codexOk   ? parseVerdict(extractCodexText(codexResult.output)) : null;
const minimaxVerdict = minimaxOk ? parseVerdict(minimaxResult.text)                   : null;

// D-01: Either BLOCK triggers a block
const shouldBlock =
  (codexVerdict   && codexVerdict.decision   === 'block') ||
  (minimaxVerdict && minimaxVerdict.decision === 'block');

// D-02: Report each model separately in the reason string
function buildBlockReason(codexVerdict, minimaxVerdict) {
  const codexPart   = codexVerdict
    ? (codexVerdict.decision   === 'block' ? 'Codex found: '   + codexVerdict.issue   : 'Codex: PASS')
    : 'Codex: skipped';
  const minimaxPart = minimaxVerdict
    ? (minimaxVerdict.decision === 'block' ? 'MiniMax found: ' + minimaxVerdict.issue : 'MiniMax: PASS')
    : 'MiniMax: skipped';
  return codexPart + '. ' + minimaxPart + '. Fix before responding.';
}
```

### Pattern: Per-Model Token Logging (D-06, D-07)

Both models log separately to `token-log.jsonl`. The `dual_review: true` flag on both records allows cost reporting to distinguish dual vs single review sessions.

```javascript
// Log Codex result (if ran)
if (codexOk && codexResult.tokens) {
  const record = {
    timestamp:        new Date().toISOString(),
    session_id:       data.session_id || null,
    model:            'gpt-5.4',
    source:           'cli',
    task_type:        'review',
    review_task_type: taskType,
    dual_review:      true,           // D-07
    verdict:          codexVerdict.decision === 'block' ? 'BLOCK' : 'ALLOW',
    block_summary:    codexVerdict.decision === 'block' ? codexVerdict.issue : null,
    tokens: { /* ... */ },
    cost_usd:         computeCost(codexResult.tokens, 'gpt-5.4'),
    rate_limit_pct:   codexResult.tokens.rate_limit_pct || null
  };
  fs.appendFileSync(logPath, JSON.stringify(record) + '\n', 'utf8');
}

// Log MiniMax result (if ran) — separate entry, same log file
if (minimaxOk && minimaxResult.tokens) {
  const record = {
    timestamp:        new Date().toISOString(),
    session_id:       data.session_id || null,
    model:            'minimax-m2.7',
    source:           'api',
    task_type:        'review',
    review_task_type: taskType,
    dual_review:      true,           // D-07
    verdict:          minimaxVerdict.decision === 'block' ? 'BLOCK' : 'ALLOW',
    block_summary:    minimaxVerdict.decision === 'block' ? minimaxVerdict.issue : null,
    tokens: {
      input:        minimaxResult.tokens.input_tokens        || 0,
      cached_input: minimaxResult.tokens.cached_input_tokens || 0,
      output:       minimaxResult.tokens.output_tokens       || 0,
    },
    cost_usd:  computeCodexCostStrict(minimaxResult.tokens, 'minimax-m2.7') || 0
  };
  fs.appendFileSync(logPath, JSON.stringify(record) + '\n', 'utf8');
}
```

### Pattern: Differentiated Review Prompts (Claude's Discretion)

The CONTEXT.md notes that identical prompts dilute the "different perspective" value. Research confirms this: MiniMax matched Opus 100% on bug and security detection but produced simpler architecture. The recommendation is to split concerns:

- **Codex prompt:** Keep the existing `buildReviewPrompt()` output — correctness, architecture, requirements alignment, convention.
- **MiniMax prompt:** Security vulnerabilities, edge cases, race conditions, input validation, error handling paths.

This means `buildReviewPrompt` needs a second variant, or an optional `focus` parameter. The simplest approach is a second function `buildMinimaxReviewPrompt(diff, taskType)` that reuses the diff but replaces the review categories. For `bulk-ops` and `test-gen` task types, the light prompt is already appropriate for both models — no differentiation needed.

### Pattern: MiniMax Diff Truncation (Claude's Discretion)

Current hook truncates at 8000 chars. MiniMax context window is 205K tokens. The binding constraint is not context size — it is prompt predictability and cost control. Recommendation: reuse the same 8000-char limit for MiniMax. This also means the existing `collectDiff()` output is passed directly to the MiniMax prompt without modification.

### Anti-Patterns to Avoid

- **Using `Promise.all` where either call can throw:** Both `runCodexExec` and `runMinimax` catch internally. If a future refactor makes them throw, switch to `Promise.allSettled`.
- **Blocking on both model failures:** If both fail, the hook must exit 0 (fail-open), never output a `decision: 'block'`.
- **Single `additionalContext` combining both pass summaries:** When both pass, output one clear summary: "Codex: PASS. MiniMax: PASS." — not two separate hook outputs.
- **Logging both models in a single JSONL record:** Each model must have its own log entry so cost aggregation and per-model reporting work correctly.
- **Changing `stop_hook_active` guard placement:** It must remain the very first check. Moving it or adding code before it risks infinite loops.

---

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| MiniMax API call | Custom HTTP client | `runMinimax()` from `minimax-exec.js` | Phase 8 already handles timeout, retry, token parsing, cost computation, and the `MINIMAX_API_KEY` guard |
| Codex rate-limit detection | Custom exit-code parsing | `isCodexRateLimited()` from `minimax-exec.js` | Covers all 4 D-15 detection methods (pct, stderr keywords, 429 in JSONL, timeout+no-output) |
| MiniMax token cost | Inline math | `computeCodexCostStrict(tokens, 'minimax-m2.7')` from `codex-pricing.js` | Pricing already added in Phase 8; strict variant returns null for unknown models |
| Verdict parsing | New parsing logic | Existing `parseVerdict()` in `codex-review-gate.js` | Already handles all-lines scan, case-insensitive match, fail-open default |

**Key insight:** Phase 9 is a composition phase. All building blocks exist. The work is wiring them together correctly — `Promise.all`, merge logic, and dual logging.

---

## Common Pitfalls

### Pitfall 1: `Promise.all` rejects on first failure when either call throws
**What goes wrong:** If `runCodexExec` or `runMinimax` ever throws (rather than returning `{ success: false }`), `Promise.all` rejects immediately, the catch block fires, and the hook exits 0 — silently skipping review.
**Why it happens:** `runCodexExec` spawns a subprocess; spawn errors are caught but library bugs can still throw. `runMinimax` wraps its API call in try/catch but the lazy-require for the OpenAI SDK can throw.
**How to avoid:** Wrap each leg in its own try/catch before passing to `Promise.all`, or use `Promise.allSettled`. The safest pattern: wrap both `runCodexExec` and `runMinimax` calls in individual promise wrappers that catch and return `{ success: false, error: e.message }`.
**Warning signs:** Hook exits 0 silently with no log entry during testing.

### Pitfall 2: MiniMax verdict parsed from JSONL instead of plain text
**What goes wrong:** `parseVerdict` is called on MiniMax output the same way it's called on Codex output. Codex output is JSONL that needs `extractCodexText()` first. MiniMax output (`minimaxResult.text`) is already plain text — calling `extractCodexText` on it would try to JSON-parse plain text lines and return empty string.
**Why it happens:** Copy-paste of the Codex verdict parsing block without noticing the difference.
**How to avoid:** For Codex: `parseVerdict(extractCodexText(codexResult.output))`. For MiniMax: `parseVerdict(minimaxResult.text)` directly. No `extractCodexText` needed.
**Warning signs:** MiniMax always returns ALLOW even when it outputs "BLOCK:" — because `extractCodexText` stripped the plain-text response.

### Pitfall 3: Token shape mismatch between Codex and MiniMax log records
**What goes wrong:** The existing logging block uses `result.tokens.input_tokens` etc. (Codex shape). MiniMax returns the same shape `{ input_tokens, cached_input_tokens, output_tokens }` from `runMinimax`, so it looks the same — but `computeCost` (backward-compat, falls back to gpt-5.4) must not be used for MiniMax. Use `computeCodexCostStrict(tokens, 'minimax-m2.7')` instead.
**Why it happens:** Copying the Codex logging block for MiniMax and not updating the cost function.
**How to avoid:** Use different cost functions per model. For Codex: `computeCost(tokens, 'gpt-5.4')`. For MiniMax: `computeCodexCostStrict(tokens, 'minimax-m2.7') || 0`.
**Warning signs:** MiniMax costs logged as if priced at gpt-5.4 rates (8x too expensive in the log).

### Pitfall 4: MiniMax `source` field inconsistency
**What goes wrong:** The Codex log record uses `source: 'cli'`. The MiniMax record should use `source: 'api'` (not `'cli'`, not `'api-fallback'`). Using `'api-fallback'` (the fallback chain value) would misattribute direct Phase 9 MiniMax calls as fallback events in cost reports.
**Why it happens:** Copying `runWithFallback` result shape, which sets `source: 'api-fallback'`.
**How to avoid:** Hardcode `source: 'api'` in the MiniMax log record for Phase 9 direct calls.
**Warning signs:** Cost aggregator shows MiniMax reviews as "fallback" events even when Codex succeeded.

### Pitfall 5: Wall-clock time exceeds hook timeout
**What goes wrong:** Both models run in parallel with 120s timeouts. The hook's outer stdin timeout is 300s. But if both models approach their 120s limit, the hook itself uses ~120s + overhead. Add hook startup, diff collection, and logging time and you approach the 300s budget.
**Why it happens:** The 300s hook timeout was sized for sequential Codex-only review. Parallel execution halves wall-clock time but the absolute time budget is unchanged.
**How to avoid:** 120s per model is appropriate — MiniMax pre-answer latency documented at up to 55s, Codex CLI at up to 120s. Total wall-clock is max(120, 120) = 120s, well within the 300s budget. No change needed.
**Warning signs:** Hook exits via stdin timeout (exits 0 after 300s with no output).

### Pitfall 6: Advisory output format for double-PASS case
**What goes wrong:** Writing two separate `additionalContext` JSON blocks to stdout when both models PASS. Claude Code may not concatenate them as expected; only one may be surfaced.
**Why it happens:** Treating each model's result as independent output.
**How to avoid:** Construct a single JSON output object with combined text: `{ additionalContext: 'Codex reviewed: PASS. MiniMax reviewed: PASS.' }`.
**Warning signs:** Only one model's PASS is visible in Claude's context after the hook runs.

---

## Code Examples

### Full Parallel Invocation Block (verified pattern)

```javascript
// Source: Node.js Promise.all docs + D-03 from CONTEXT.md
// Both calls are wrapped to guarantee they never throw — Promise.all stays safe

async function runParallelReviews(codexPrompt, minimaxPrompt, cwd) {
  const [codexResult, minimaxResult] = await Promise.all([
    // Codex leg: direct call (not runWithFallback — keeps the two legs independent)
    runCodexExec(codexPrompt, { cwd, timeoutMs: 120000, model: 'gpt-5.4' })
      .catch((e) => ({ success: false, output: '', tokens: null, error: e.message })),

    // MiniMax leg: direct API call
    runMinimax(minimaxPrompt, { maxTokens: 2000, timeoutMs: 120000 })
      .catch((e) => ({ success: false, text: '', tokens: null, cost: 0, error: e.message }))
  ]);

  return { codexResult, minimaxResult };
}
```

### Differentiated Prompt Builder (Claude's Discretion recommendation)

```javascript
// Codex prompt: reuse existing buildReviewPrompt() unchanged
const codexPrompt = buildReviewPrompt(diff, taskType);

// MiniMax prompt: security/edge-case focus (new function)
function buildMinimaxReviewPrompt(diff, taskType) {
  if (taskType === 'bulk-ops' || taskType === 'test-gen') {
    // Light review — same as Codex light prompt
    return buildReviewPrompt(diff, taskType);
  }
  return `Review the following code changes for security and edge-case issues. Output your verdict as the FIRST line:
- "ALLOW" if the changes are acceptable
- "BLOCK: <one-line reason>" if there is a significant issue

Review specifically for:
1. Security vulnerabilities (injection, authentication bypass, insecure data handling)
2. Edge cases and boundary conditions (null/undefined handling, empty inputs, overflow)
3. Race conditions or concurrency hazards
4. Error handling gaps (uncaught exceptions, silent failures)

Only BLOCK for significant issues. Minor observations should ALLOW with a note.

Changes:
${diff}`;
}
```

### Complete Verdict Merge and Output

```javascript
// After Promise.all resolves:
const codexOk   = codexResult.success;
const minimaxOk = minimaxResult.success;

// Both failed — fail-open
if (!codexOk && !minimaxOk) {
  process.stderr.write('codex-review-gate: both models failed — skipping review\n');
  process.exit(0);
}

// Parse verdicts (only for successful runs)
const codexVerdict   = codexOk   ? parseVerdict(extractCodexText(codexResult.output)) : null;
const minimaxVerdict = minimaxOk ? parseVerdict(minimaxResult.text)                   : null;

// D-01: block if either flags an issue
const shouldBlock =
  (codexVerdict   && codexVerdict.decision   === 'block') ||
  (minimaxVerdict && minimaxVerdict.decision === 'block');

if (shouldBlock) {
  const codexPart = codexVerdict
    ? (codexVerdict.decision === 'block' ? 'Codex found: ' + codexVerdict.issue : 'Codex: PASS')
    : 'Codex: skipped';
  const minimaxPart = minimaxVerdict
    ? (minimaxVerdict.decision === 'block' ? 'MiniMax found: ' + minimaxVerdict.issue : 'MiniMax: PASS')
    : 'MiniMax: skipped';
  process.stdout.write(JSON.stringify({
    decision: 'block',
    reason: codexPart + '. ' + minimaxPart + '. Fix before responding.'
  }));
} else {
  const passedModels = [codexOk ? 'Codex' : null, minimaxOk ? 'MiniMax' : null]
    .filter(Boolean).join(' + ');
  process.stdout.write(JSON.stringify({
    additionalContext: passedModels + ' reviewed: PASS'
  }));
}
```

---

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Sequential single-model review | Parallel dual-model review | Phase 9 | Wall-clock time unchanged (parallel); two independent verdicts; catch rate increases |
| Codex-only PASS advisory | Named model PASS summary | Phase 9 | User sees which models reviewed, not just a generic PASS |
| Single token log entry per review | Two entries per review | Phase 9 | Cost aggregator can compute dual vs single review costs; per-model catch rate tracking |

---

## Open Questions

1. **Should the Codex leg use `runCodexExec` directly or `runWithFallback`?**
   - What we know: `runWithFallback` provides rate-limit escalation to MiniMax. But in Phase 9, MiniMax is already running in the parallel leg. If Codex is rate-limited and `runWithFallback` escalates to MiniMax, MiniMax runs twice — once as the fallback for Codex, once as the direct parallel leg.
   - What's unclear: Whether double-MiniMax spend on rate-limit is acceptable vs having a "Codex: skipped" result.
   - Recommendation: Use `runCodexExec` directly for the Codex leg. On rate-limit, Codex fails (`success: false`), MiniMax leg still ran independently, result is MiniMax-only review. This is clean separation and avoids double MiniMax cost.

2. **What `source` field value should MiniMax direct-call log entries use?**
   - What we know: Existing values are `'cli'` (Codex), `'api-fallback'` (fallback chain), `'api'` (runGpt54MiniApi).
   - Recommendation: Use `'api'` for direct MiniMax calls in Phase 9. Reserve `'api-fallback'` for the fallback chain path in `runWithFallback`.

---

## Environment Availability

| Dependency | Required By | Available | Version | Fallback |
|------------|------------|-----------|---------|----------|
| Node.js | Hook execution | Yes | v22.22.0 | — |
| `codex` CLI | Codex review leg | Yes | 0.118.0 | MiniMax-only review |
| `minimax-exec.js` | MiniMax review leg | Yes | Phase 8 output | Codex-only review |
| `openai` npm package | `runMinimax` internal dep | Yes | v6.33.0 at global path | — |
| `MINIMAX_API_KEY` env var | `runMinimax` | **NOT SET** | — | Hook fails open (MiniMax leg returns `success: false`); Codex-only review proceeds |
| `OPENAI_API_KEY` env var | Codex CLI | Yes (set) | — | — |
| `codex-review-gate.js` (Stop hook) | This phase modifies it | Yes | v2.0.0 | — |
| Stop hook registration in `~/.claude/settings.json` | Hook fires on Stop events | Yes | Registered | — |

**Missing dependencies with no fallback:**
- None that block hook operation.

**Missing dependencies with fallback:**
- `MINIMAX_API_KEY` not set in current shell environment. This means MiniMax leg will return `{ success: false, error: 'MINIMAX_API_KEY is not set' }` during testing. The fail-open path handles this correctly — review falls back to Codex-only. The API key must be set in the user's shell profile for the hook to function in dual-review mode. The Phase 8 connectivity test script (`minimax-connectivity-test.js`) should be run after the API key is configured to verify end-to-end function before relying on dual review in production.

---

## Sources

### Primary (HIGH confidence)

- `/home/alucard/.claude/hooks/minimax-exec.js` — Phase 8 output, directly read. `runMinimax()` signature, return shape `{ success, text, tokens, cost, error }`, token field names.
- `/home/alucard/.claude/hooks/codex-review-gate.js` — Directly read. Full Stop hook implementation, existing patterns for verdict parsing, token logging, fail-open behavior.
- `/home/alucard/.claude/hooks/codex-exec.js` — Directly read. `runCodexExec()` signature, return shape `{ success, exitCode, output, tokens, rateLimitPct, error }`.
- `/home/alucard/.claude/hooks/codex-pricing.js` — Directly read. `computeCost`, `computeCodexCostStrict` signatures, `minimax-m2.7` pricing entry confirmed present.
- `/home/alucard/projects/Claude_X_Codex/minimax-m2.7-synthesis.md` — Directly read. Section 5 critical gotchas, section 6 hook integration patterns.
- `.planning/phases/09-dual-review-gate/09-CONTEXT.md` — Directly read. All locked decisions D-01 through D-07.
- Node.js v22 `Promise.all` — Stable built-in; behavior has not changed since Node.js 4.

### Secondary (MEDIUM confidence)

- `minimax-m2.7-synthesis.md` §2 benchmarks — MiniMax bug/security detection matching Opus 100% in head-to-head testing (independent testing, not self-reported).
- MiniMax pre-answer latency up to 55s — Documented in synthesis §4 (multiple sources agree).

### Tertiary (LOW confidence)

- None — all critical claims are verified against source files on disk.

---

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH — all modules read directly, versions confirmed
- Architecture patterns: HIGH — verified against actual function signatures and return shapes in source files
- Pitfalls: HIGH — derived from direct code inspection, not from external sources
- Environment availability: HIGH — verified by live shell commands

**Research date:** 2026-04-03
**Valid until:** Stable (code is local; no external API changes affect hook internals)
