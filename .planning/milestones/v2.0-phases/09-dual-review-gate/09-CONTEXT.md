# Phase 9: Dual Review Gate - Context

**Gathered:** 2026-04-03
**Status:** Ready for planning

<domain>
## Phase Boundary

Modify the Stop hook (`codex-review-gate.js`) to run Codex and MiniMax reviews in parallel. Both models independently review code changes, and their verdicts are merged. This adds a genuinely different perspective to catch issues either model alone would miss.

</domain>

<decisions>
## Implementation Decisions

### Verdict merge strategy
- **D-01:** Either model blocks — if EITHER Codex or MiniMax flags an issue, the response is BLOCKED. Most conservative approach. Matches the project's core value: "cross-model review catches what any single model misses alone."
- **D-02:** Both verdicts reported separately in the block reason. Output format: "Codex found: [issue]. MiniMax found: [issue]." or "Codex: PASS. MiniMax found: [issue]." so Opus knows which model flagged what.

### Parallel execution
- **D-03:** Use `Promise.all` to run Codex CLI (`runCodexExec`) and MiniMax API (`runMinimax`) simultaneously. Total wall-clock time is the slower of the two, not additive.
- **D-04:** If Codex is rate-limited, MiniMax review still runs independently (graceful degradation via `runWithFallback` from Phase 8). The review becomes single-model, not zero-model.
- **D-05:** If MiniMax fails (API error, timeout), Codex review still proceeds. MiniMax failure is fail-open — the existing Codex-only behavior is the fallback.

### Token logging
- **D-06:** Log both models' reviews as separate entries in `token-log.jsonl`. Fields: `model: 'gpt-5.4'` and `model: 'minimax-m2.7'`, both with `task_type: 'review'`, `review_task_type: [feature|security|test-gen|bulk-ops]`.
- **D-07:** Track `dual_review: true` flag in log entries so cost reporting can show dual vs single review costs.

### Claude's Discretion
- Timeout values for MiniMax review call (suggest matching Codex's 120s)
- How to truncate diff for MiniMax if it exceeds context limits (MiniMax has 205K vs Codex's larger window)
- Review prompt adaptation for MiniMax (same prompt as Codex or tailored)

</decisions>

<specifics>
## Specific Ideas

- The review prompt for MiniMax should emphasize different concerns than Codex — if both use identical prompts, the "different perspective" value is diluted. Consider Codex focusing on correctness/architecture and MiniMax focusing on edge cases/security.
- Keep the existing `stop_hook_active` infinite loop guard — it must still prevent review-triggers-review loops.

</specifics>

<canonical_refs>
## Canonical References

### Phase 8 Foundation (dependency)
- `.planning/phases/08-minimax-foundation/08-CONTEXT.md` — Module architecture, `runMinimax()` interface, `runWithFallback()` fallback chain

### Existing Review Gate
- `~/.claude/hooks/codex-review-gate.js` — Current single-model review implementation to modify
- `~/.claude/hooks/codex-exec.js` — `runCodexExec()` function used for Codex review invocation

### Research
- `minimax-m2.7-synthesis.md` §5 "Critical Gotchas" — MiniMax-specific limitations to handle
- `minimax-m2.7-synthesis.md` §6 "Hook Integration" — Proposed Stop hook changes

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `codex-review-gate.js` — Complete Stop hook with diff collection, task classification, verdict parsing, token logging. Only the invocation and verdict merge need modification.
- `minimax-exec.js` (from Phase 8) — `runMinimax(prompt, opts)` for the MiniMax API call.

### Established Patterns
- Verdict parsing scans ALL output lines for ALLOW/BLOCK pattern
- Token logging appends JSONL with consistent schema
- Fail-open on infrastructure errors (never block on hook failure)
- `stop_hook_active` flag prevents infinite loops

### Integration Points
- `codex-review-gate.js` currently calls `runCodexExec(reviewPrompt, { cwd, timeoutMs: 120000, model: 'gpt-5.4' })`. Add parallel `runMinimax(reviewPrompt, { maxTokens: 2000 })`.
- Verdict merge happens after both calls resolve.

</code_context>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope.

</deferred>

---

*Phase: 09-dual-review-gate*
*Context gathered: 2026-04-03*
