# Phase 10: Adversarial Plan Review - Research

**Researched:** 2026-04-03
**Domain:** Multi-round plan review orchestration, MiniMax M-2.7 reasoning API, Node.js OpenAI SDK
**Confidence:** HIGH

---

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions

- **D-01:** Modify `codex-multi-round-reviewer.js` to accept a `model` parameter per round. Round 1: Codex (constructive). Round 2: MiniMax (adversarial). The round config becomes `[{ model: 'codex', type: 'constructive' }, { model: 'minimax', type: 'adversarial' }]`.
- **D-02:** Applies to BOTH `codex-plan-reviewer.js` (GSD) and `codex-superpowers-plan-reviewer.js` (Superpowers). Same multi-round reviewer module, same config.
- **D-03:** Preserve everything. MiniMax's full `<think>` reasoning chain appears in REVIEWS.md output. User sees MiniMax's complete thought process alongside its findings. Full transparency.
- **D-04:** Use `reasoning_split: true` in the MiniMax API call (via `extra_body`) to get structured reasoning in the `reasoning_details` field. Preserve BOTH the structured field AND any inline `<think>` tags in conversation history.
- **D-05:** When passing Round 1 Codex findings to MiniMax for Round 2, include the full findings text. MiniMax receives: "Here are the constructive review findings from Round 1: [Codex output]. Now conduct an adversarial review — poke holes, find flaws, challenge assumptions, act as devil's advocate."
- **D-06:** `review-state.json` updated to track which model ran each round: `rounds[].model` field added (`'gpt-5.4'` or `'minimax-m2.7'`).
- **D-07:** Token logging per round tracks the correct model. Round 1 logged as `model: 'gpt-5.4'`, Round 2 as `model: 'minimax-m2.7'`.
- **D-08:** If MiniMax is unavailable for Round 2, fall back to Codex adversarial (current behavior). The plan review degrades to same-model-twice rather than failing entirely. Log the fallback event.

### Claude's Discretion

- Adversarial prompt wording (how aggressively MiniMax should challenge)
- Whether to add a Round 3 synthesis option in the future (deferred)
- Exact format of `<think>` content in REVIEWS.md (inline vs collapsible section)

### Deferred Ideas (OUT OF SCOPE)

- Round 3 synthesis (merge R1+R2 into unified assessment) — evaluate after v2.0 ships
- Using MiniMax for BOTH rounds when Codex is rate-limited (already handled by D-08 fallback)
</user_constraints>

---

## Summary

Phase 10 modifies a single shared module (`codex-multi-round-reviewer.js`) to route Round 2 of the multi-round plan review to MiniMax M-2.7 instead of Codex. The change is architecturally small but has two non-trivial integration points: (1) wiring `reasoning_split: true` into the MiniMax API call so the full `<think>` chain is captured and preserved in REVIEWS.md, and (2) feeding Round 1 Codex findings as context into the Round 2 MiniMax adversarial prompt.

The Phase 8 foundation (`minimax-exec.js`) is fully implemented and verified. `runMinimax(prompt, opts)` is ready to call; it needs a `reasoning_split: true` flag added to the API request. The existing round loop, state management, and token logging in `codex-multi-round-reviewer.js` v3.0 require targeted modifications — not a rewrite. The callers (`codex-plan-reviewer.js` and `codex-superpowers-plan-reviewer.js`) need zero changes; they call `runMultiRoundReview()` which hides the model routing internally.

The adversarial prompt should be explicitly red-team framed — not just "find issues" but "break this plan." MiniMax's self-evolution reasoning style makes it a qualitatively different adversary than Codex reviewing the same material twice.

**Primary recommendation:** Add `reasoning_split: true` opt-in to `runMinimax()`, feed Round 1 findings as preamble context to Round 2, and update round state records to include `model` field. Three targeted edits to one file.

---

## Standard Stack

### Core (All Already Installed)

| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| `openai` npm package | 6.33.0 | MiniMax API calls via OpenAI SDK + baseURL swap | Already used in minimax-exec.js; zero new deps |
| Node.js | 22.22.0 | Runtime for hook scripts | All existing hooks use Node.js |
| `codex-exec.js` | v3.0 (installed) | Round 1 Codex invocation | Unchanged from current implementation |
| `minimax-exec.js` | v1.0.0 (installed) | Round 2 MiniMax invocation | Phase 8 foundation; ready to call |

### Supporting (Already Present)

| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| `codex-pricing.js` | v1.0.0 | Cost computation for MiniMax token logging | Used by minimax-exec.js via computeCodexCostStrict |
| Node.js `fs` | built-in | State file read/write | review-state.json persistence |
| Node.js `path` | built-in | Phase directory resolution | HANDOFF.md and REVIEWS.md path construction |

### Alternatives Considered

| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| Extending `runMinimax()` | New `runMinimaxWithReasoning()` | Extending is simpler — single flag, backward-compat |
| Inline reasoning_split in multi-round-reviewer | Passing via opts to runMinimax | Passing via opts is cleaner; caller controls the flag |

**Installation:** No new packages required. All dependencies are installed.

---

## Architecture Patterns

### What Changes and What Does Not

```
codex-multi-round-reviewer.js (MODIFY — targeted changes only)
├── Round 1: runCodexExec()  → UNCHANGED
├── Round 2: runCodexExec()  → REPLACE with runMinimax()
│   ├── Add: require('./minimax-exec') at top of file
│   ├── Add: reasoning_split: true in runMinimax opts
│   ├── Add: Round 1 findings as preamble to Round 2 prompt
│   └── Add: D-08 fallback — if runMinimax fails, retry with runCodexExec
├── recordRoundResult(): add model field to round record (D-06)
├── logTokens(): accept model param; use 'minimax-m2.7' for Round 2 (D-07)
└── Everything else: UNCHANGED

codex-plan-reviewer.js          → NO CHANGES (calls runMultiRoundReview())
codex-superpowers-plan-reviewer.js → NO CHANGES (calls runMultiRoundReview())
minimax-exec.js                 → MINOR CHANGE: add reasoning_split support
```

### Pattern 1: reasoning_split via Body Spread (OpenAI SDK v6)

**What:** MiniMax-specific `reasoning_split: true` is passed by spreading it directly into the first argument of `client.chat.completions.create()`. The SDK serializes the entire first arg as the JSON request body, so any extra fields go through to the API.

**Why not `extra_body` (Python SDK syntax):** In Node.js OpenAI SDK v6, `create(body, options)` — the first arg IS the body. The `RequestOptions` second arg has a `body` field but it replaces the whole body, not merges. Spreading into the first arg is the correct Node.js pattern.

**Verified:** SDK source (`completions.js` line: `return this._client.post('/chat/completions', { body, ...options, stream: body.stream ?? false })`) confirms the first arg becomes the JSON body verbatim.

```javascript
// Source: minimax-exec.js (verified from installed package)
// Add reasoning_split opt-in to runMinimax()
const response = await callWithRetry(async () => {
  const requestBody = {
    model,
    messages,
    max_tokens: maxTokens,
    temperature,
  };
  if (reasoningSplit) {
    requestBody.reasoning_split = true;
  }
  return await client.chat.completions.create(requestBody, { signal: controller.signal });
});
```

### Pattern 2: Extracting reasoning_details from MiniMax Response

**What:** When `reasoning_split: true`, MiniMax returns the thinking chain in `response.choices[0].message.reasoning_content` (the field name may vary — MiniMax docs show `reasoning_details` while the actual API may use `reasoning_content`). The `content` field contains only the final answer. Both must be captured.

**Confidence:** MEDIUM — field name requires runtime verification. The synthesis doc says `reasoning_details`; the CONTEXT.md says `reasoning_details`. Implementation should defensively check both.

```javascript
// Defensive extraction pattern for MiniMax reasoning fields
const message = response.choices[0].message;
const finalContent = message.content || '';
const reasoningContent = message.reasoning_content      // primary MiniMax field
                      || message.reasoning_details      // alternative field name
                      || '';

// Combined text for REVIEWS.md (D-03: preserve full transparency)
const fullText = reasoningContent
  ? `<think>\n${reasoningContent}\n</think>\n\n${finalContent}`
  : finalContent;
```

### Pattern 3: Round 1 Context Passing to Round 2 (D-05)

**What:** The full Round 1 Codex findings text is prepended to the Round 2 MiniMax adversarial prompt. MiniMax receives the plan AND what Codex already found, with explicit instruction to act as devil's advocate — free to disagree with Codex.

```javascript
// In runMultiRoundReview(), Round 2 prompt construction
const round2Prompt = round1Text
  ? `Here are the constructive review findings from Round 1 (by Codex):\n\n${round1Text}\n\n---\n\nYou are NOT constrained by the above. Your job is adversarial: poke holes, find flaws, challenge assumptions. Act as devil's advocate.\n\n${ADVERSARIAL_PROMPT}${planContent}`
  : ADVERSARIAL_PROMPT + planContent;
```

### Pattern 4: D-08 Fallback — Graceful MiniMax Failure

**What:** If `runMinimax()` returns `{ success: false }` for Round 2, retry with `runCodexExec()` using the same adversarial prompt (current behavior). Log the fallback event to stderr and to the round record.

```javascript
// Round 2 with fallback
let r2Result = await runMinimax(round2Prompt, { ... reasoning_split: true ... });
let r2Model = 'minimax-m2.7';

if (!r2Result.success) {
  process.stderr.write(`codex-multi-round: MiniMax unavailable for Round 2 — falling back to Codex\n`);
  r2Result = await runCodexExec(round2Prompt, { cwd, timeoutMs: 180000, model: 'gpt-5.4' });
  r2Model = 'gpt-5.4';
  // r2Result.text = extractCodexText(r2Result.output) if Codex path
}
```

### Pattern 5: review-state.json model field (D-06)

**What:** Add `model` field to each round record in `review-state.json`. This is additive — existing code that reads `round.review_type` or `round.text` is unaffected.

```javascript
// recordRoundResult() — add model field
const roundRecord = {
  round: roundNum,
  review_type: result.reviewType,
  model: result.model,           // 'gpt-5.4' | 'minimax-m2.7' — NEW (D-06)
  completed_at: new Date().toISOString(),
  issue_count: result.issueCount,
  has_high_issues: result.hasHighIssues,
  early_exit: result.earlyExit || false,
  text: result.text
};
```

### Pattern 6: Token Logging with Correct Model (D-07)

**What:** `logTokens()` currently hardcodes `model: 'gpt-5.4'` and uses `computeCost()`. For Round 2 it must use `model: 'minimax-m2.7'` and `computeCodexCostStrict()`. The function signature needs a `model` parameter.

```javascript
// logTokens() — add model parameter
function logTokens(cwd, sessionId, roundNum, reviewType, result, model) {
  const modelId = model || 'gpt-5.4';
  const { computeCodexCostStrict, computeCost } = require('./codex-pricing');
  // For minimax, use computeCodexCostStrict; for codex, existing computeCost works
  const cost = modelId === 'minimax-m2.7'
    ? (computeCodexCostStrict(result.tokens, 'minimax-m2.7') || 0)
    : computeCost(result.tokens, modelId);

  const record = {
    // ...
    model: modelId,               // D-07: correct model per round
    source: modelId === 'minimax-m2.7' ? 'api' : 'cli',
    // ...
    cost_usd: cost,
  };
}
```

### Anti-Patterns to Avoid

- **Stripping `<think>` from Round 2 text:** MiniMax quality degrades silently if reasoning is stripped from conversation history. Never filter `<think>` content before or after the API call (D-03, synthesis §5 item 1).
- **Using runWithFallback() for Round 2:** `runWithFallback()` triggers MiniMax on Codex rate-limit, which is the Phase 9 pattern. Phase 10 wants MiniMax as primary for Round 2, so call `runMinimax()` directly. D-08 fallback is Codex, not the other direction.
- **Passing Round 1 findings as a system message:** MiniMax works better with the context in the user message — mixing system/user roles for plan review can degrade output quality. Keep the handoff in the user-role message.
- **Hardcoding reasoning_split in runMinimax():** Not every caller wants reasoning overhead. Make it opt-in via `opts.reasoningSplit = true` so other callers are unaffected.
- **Temperature = 0 for MiniMax:** MiniMax API rejects `temperature: 0` exactly. Use `0.01` (already set in minimax-exec.js, D-03 of Phase 8).

---

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| MiniMax API client | Custom HTTP fetch wrapper | `runMinimax()` from minimax-exec.js | Already implemented with retry, AbortController, token extraction |
| Codex JSONL parsing | Custom JSONL parser | `extractCodexText()` (already in multi-round-reviewer.js) | Handles all Codex output format variations |
| Token cost computation | Custom pricing math | `computeCodexCostStrict()` / `computeCost()` from codex-pricing.js | Centralized, tested, used by all other hooks |
| State persistence | Custom state format | Existing `loadOrInitState()` / `recordRoundResult()` | Resume logic, schema versioning, error handling already built |
| Fallback detection | Custom retry logic | Call `runCodexExec()` directly in the fallback branch | D-08 is simple: MiniMax fails → try Codex. No complex detection needed |

**Key insight:** This phase is a targeted extension of existing infrastructure. The heavy lifting — API calls, token logging, state management, HANDOFF.md writing — is already done. Phase 10 only routes the Round 2 call to a different model.

---

## Common Pitfalls

### Pitfall 1: reasoning_content vs reasoning_details Field Name

**What goes wrong:** MiniMax API may return the thinking chain in `reasoning_content` or `reasoning_details` depending on API version. Code that hardcodes one field name silently loses the other.

**Why it happens:** MiniMax documentation and the synthesis research use different field names. The API is relatively new and field naming was inconsistent across docs.

**How to avoid:** Check both fields defensively:
```javascript
const thinking = message.reasoning_content || message.reasoning_details || '';
```

**Warning signs:** Empty `<think>` sections in REVIEWS.md despite MiniMax returning a response.

### Pitfall 2: Round 2 Text Extraction Is Different for MiniMax vs Codex

**What goes wrong:** `extractCodexText()` parses Codex JSONL output format. MiniMax returns a plain `response.choices[0].message.content` string (not JSONL). Passing MiniMax output through `extractCodexText()` will return empty string.

**Why it happens:** Round 1 uses Codex CLI which emits JSONL; Round 2 uses MiniMax SDK which returns a parsed object. The existing `extractCodexText()` function is JSONL-specific.

**How to avoid:** For Round 2, use `r2Result.text` directly from `runMinimax()` return value (already a plain string). Only call `extractCodexText()` on Codex CLI raw output.

### Pitfall 3: Verbosity Tax Bloating review-state.json and REVIEWS.md

**What goes wrong:** MiniMax generates ~4x more output tokens than median models. With `reasoning_split: true`, the full `<think>` chain adds further output. REVIEWS.md becomes very large; `review-state.json` `text` field swells.

**Why it happens:** MiniMax's verbosity is a known characteristic (synthesis §3). The thinking chain can be substantial — it shows HOW the model reasoned, not just WHAT it found.

**How to avoid:** Set `maxTokens` for Round 2 higher than the default 2000 (e.g., 4000–6000) to accommodate both reasoning and findings, but not unlimited. The plan REVIEWS.md growing large is expected and acceptable — it's the transparency goal of D-03.

### Pitfall 4: D-08 Fallback Logic Must Match runMinimax() Return Shape

**What goes wrong:** `runMinimax()` returns `{ success, text, tokens, cost, error }`. `runCodexExec()` returns `{ success, output, tokens, error }` — note `output` vs `text`. The fallback branch must normalize to a common shape before calling `extractCodexText()` on the Codex path.

**Why it happens:** The two model modules have slightly different return shapes (established in Phase 8 and Phase 9).

**How to avoid:** In Round 2, normalize early:
```javascript
// After MiniMax call:
round2Text = r2Result.text;                     // MiniMax path

// After Codex fallback call:
round2Text = extractCodexText(r2Result.output);  // Codex path
```

### Pitfall 5: Early Exit Check Must Run Before Round 2 Prompt Construction

**What goes wrong:** If Round 1 returns 0 issues (early exit), Round 2 should be skipped entirely — but if Round 1 findings text is empty and gets included as context in the Round 2 prompt, MiniMax receives a confusing "here are the Round 1 findings: [empty]" preamble.

**Why it happens:** The early exit check already exists but prompt construction might run before the check.

**How to avoid:** The existing `earlyExit` flag in the Round 1 block already gates Round 2 execution. The Round 2 prompt construction code should be inside the `!earlyExit` block, not outside it. This is already the case in v3.0 — just ensure the preamble injection respects the same guard.

### Pitfall 6: State Resume Ignores Model Field on Old Records

**What goes wrong:** Existing `review-state.json` files from previous phases lack the `model` field on round records. If a review resumes from an old state file, `r1Record.model` is undefined, which could confuse downstream logging.

**Why it happens:** `review-state.json` is not versioned beyond `schema_version: 1`, and the existing Phase 9 state file has no `model` field on round records.

**How to avoid:** When reading resumed round records, default model to `'gpt-5.4'` if the field is absent:
```javascript
const resumedModel = r1Record.model || 'gpt-5.4';
```

---

## Code Examples

Verified patterns from official sources and installed code:

### How OpenAI SDK v6 Sends Extra Body Fields (Verified from Source)

```javascript
// Source: /home/alucard/.npm-global/lib/node_modules/openai/resources/chat/completions/completions.js
// create(body, options) → this._client.post('/chat/completions', { body, ...options })
// The ENTIRE first arg becomes the JSON body. Extra fields go through to the API.

// Correct Node.js SDK v6 pattern for reasoning_split:
const response = await client.chat.completions.create(
  {
    model: 'MiniMax-M2.7',
    messages,
    max_tokens: maxTokens,
    temperature: 0.01,
    reasoning_split: true,   // spread directly into body — not via extra_body
  },
  { signal: controller.signal }   // RequestOptions second arg
);
```

### Extracting Reasoning + Final Answer from MiniMax Response

```javascript
// Source: minimax-m2.7-synthesis.md §4 + defensive field-name handling
const message = response.choices[0].message;
const finalAnswer  = message.content || '';
const thinkingChain = message.reasoning_content   // primary field name (API)
                   || message.reasoning_details   // alternative (docs)
                   || '';

// Combine for REVIEWS.md (D-03: full transparency)
const fullReviewText = thinkingChain
  ? `<think>\n${thinkingChain}\n</think>\n\n${finalAnswer}`
  : finalAnswer;
```

### Round 2 Prompt with Round 1 Context (D-05)

```javascript
// Pass Round 1 Codex findings as context; MiniMax is free to disagree
function buildAdversarialPromptWithContext(round1FindingsText, planContent) {
  const preamble = round1FindingsText
    ? `## Round 1 Findings (constructive, by Codex)\n\n${round1FindingsText}\n\n---\n\nYou are NOT bound by the above. Treat the above as additional context, not constraints.\n\n`
    : '';
  return preamble + ADVERSARIAL_PROMPT + planContent;
}
```

### Token Logging for Round 2 (D-07)

```javascript
// logTokens() call for MiniMax Round 2
// r2Result shape from runMinimax(): { success, text, tokens, cost, error }
// tokens shape: { input_tokens, cached_input_tokens, output_tokens }
logTokens(cwd, sessionId, 2, 'adversarial', {
  tokens: r2Result.tokens,
  model: 'minimax-m2.7',    // D-07
  source: 'api',
  cost_usd: r2Result.cost,
});
```

### REVIEWS.md Header Update

```javascript
// writeReviewsFile() in codex-plan-reviewer.js needs no change to its signature.
// The review text passed to it already includes the formatted Round 2 content.
// But the header currently says "Model: gpt-5.4" — update to reflect two models:
const content = `# Phase ${paddedPhase} — Plan Review

**Reviewed:** ${timestamp}
**Models:** Round 1: gpt-5.4 (constructive), Round 2: minimax-m2.7 (adversarial)
**Plans reviewed:** ${planList}
**Review type:** Multi-round (constructive + adversarial cross-model)

## Findings

${reviewText}

## Verdict

${verdict}
`;
```

---

## Runtime State Inventory

This phase does not rename, rebrand, or migrate. The only state touched is `review-state.json` — the format change (adding `model` field to round records) is additive and backward-compatible. Existing records without the field default to `'gpt-5.4'`.

**Nothing to migrate.** Existing `review-state.json` files from Phases 9 and earlier will continue to work without modification.

---

## Environment Availability

| Dependency | Required By | Available | Version | Fallback |
|------------|------------|-----------|---------|----------|
| `minimax-exec.js` | Round 2 model | Available | v1.0.0 (Phase 8) | D-08: fall back to Codex |
| `MINIMAX_API_KEY` env var | runMinimax() | Available (set in Phase 8) | — | D-08: fall back to Codex |
| MiniMax API `https://api.minimax.io/v1` | Round 2 call | Available (verified in Phase 8) | — | D-08: fall back to Codex |
| `codex-exec.js` `runCodexExec()` | Round 1 + D-08 fallback | Available | v3.0 | — |
| `codex-pricing.js` `computeCodexCostStrict` | D-07 token logging | Available | v1.0.0 | — |
| `@openai/codex` CLI | Round 1 Codex review | Available | 0.118.0 | — |
| `openai` npm | minimax-exec.js SDK | Available | 6.33.0 | — |

**Missing dependencies with no fallback:** None.

**Missing dependencies with fallback:** If MiniMax API is unavailable at runtime, D-08 ensures Round 2 falls back to Codex adversarial review (current v3.0 behavior).

---

## State of the Art

| Old Approach (v3.0) | New Approach (Phase 10) | Impact |
|---------------------|------------------------|--------|
| Both rounds use `runCodexExec()` | Round 1: Codex, Round 2: `runMinimax()` | Genuine adversarial perspective from different reasoning style |
| Round records have no `model` field | `rounds[].model` added to state | Audit trail — know which model ran which round |
| Token logging hardcodes `model: 'gpt-5.4'` | `logTokens()` accepts model param | Round 2 cost tracked as minimax-m2.7 ($0.30/$1.20 vs $2.50/$10) |
| No reasoning chain in REVIEWS.md | Full `<think>` chain in REVIEWS.md | Transparency — see HOW MiniMax reasoned, not just what it found |
| Adversarial prompt has no Round 1 context | Round 1 findings prepended to Round 2 prompt | MiniMax can build on or challenge Codex's observations |

**Deprecated/outdated in this file after Phase 10:**
- `model: 'gpt-5.4'` hardcoded in `logTokens()` — replaced by model parameter

---

## Open Questions

1. **reasoning_content vs reasoning_details field name**
   - What we know: synthesis doc says `reasoning_details`; CONTEXT.md says `reasoning_details`; MiniMax API may use `reasoning_content`
   - What's unclear: exact field name in the API response (not testable without a live API call)
   - Recommendation: check both fields defensively in implementation; a live test call during Phase 10 verification will confirm

2. **Max tokens for Round 2 with reasoning_split**
   - What we know: default is 2000 (Phase 8 D-04); reasoning chain adds tokens; verbosity tax is ~4x
   - What's unclear: how many tokens does a typical `<think>` chain consume for plan review
   - Recommendation: start at 4000 tokens for Round 2 (2x default); adjust based on REVIEWS.md output size during verification

3. **REVIEWS.md header model field in callers**
   - What we know: `codex-plan-reviewer.js` and `codex-superpowers-plan-reviewer.js` write REVIEWS.md headers with hardcoded `Model: gpt-5.4`
   - What's unclear: should callers be updated to reflect "Round 1: gpt-5.4, Round 2: minimax-m2.7" or leave unchanged?
   - Recommendation: update the header in both callers — it's a one-line string change and improves transparency (D-03). The CONTEXT.md says D-02 applies to both callers but does not explicitly require header updates; Claude's discretion.

---

## Sources

### Primary (HIGH confidence)

- Installed source at `/home/alucard/.claude/hooks/codex-multi-round-reviewer.js` — current v3.0 implementation read in full
- Installed source at `/home/alucard/.claude/hooks/minimax-exec.js` — Phase 8 foundation implementation read in full
- Installed source at `/home/alucard/.claude/hooks/codex-plan-reviewer.js` — GSD plan reviewer read in full
- Installed source at `/home/alucard/.claude/hooks/codex-superpowers-plan-reviewer.js` — Superpowers reviewer read in full
- Installed source at `/home/alucard/.claude/hooks/codex-pricing.js` — pricing module read in full
- Installed source at `/home/alucard/.npm-global/lib/node_modules/openai/resources/chat/completions/completions.js` — SDK body serialization verified (`create(body, options)` pattern confirmed)
- Installed source at `/home/alucard/.npm-global/lib/node_modules/openai/internal/request-options.d.ts` — `RequestOptions` type confirmed (no `extra_body` field in v6)
- `.planning/phases/10-adversarial-plan-review/10-CONTEXT.md` — locked decisions read in full
- `.planning/review-state.json` — live state schema verified (schema_version: 1, no model field on round records)

### Secondary (MEDIUM confidence)

- `minimax-m2.7-synthesis.md` — MiniMax API spec, `reasoning_split` field, `reasoning_details` naming, verbosity tax, temperature constraints — verified against multiple research reports in the file
- `research/deep-research-report(3).md` — multi-model review patterns, MiniMax code review behavior, reasoning continuity requirements
- `.planning/phases/08-minimax-foundation/08-CONTEXT.md` — `runMinimax()` interface spec confirmed matches installed implementation

### Tertiary (LOW confidence)

- `reasoning_content` vs `reasoning_details` field name — could not verify from installed code; requires live API call to confirm. Defensive check pattern recommended.

---

## Metadata

**Confidence breakdown:**

- Standard stack: HIGH — all dependencies installed and verified; no new packages
- Architecture: HIGH — existing code fully read; modification points are specific and unambiguous
- Pitfalls: HIGH — extracted from live code review and synthesis research, not speculation
- `reasoning_split` mechanism: HIGH — OpenAI SDK v6 body-spread pattern confirmed from source
- `reasoning_content` field name: MEDIUM — documented in synthesis research, not runtime-verified

**Research date:** 2026-04-03
**Valid until:** 2026-05-03 (stable stack — minimax-exec.js and codex-multi-round-reviewer.js are not fast-moving)
