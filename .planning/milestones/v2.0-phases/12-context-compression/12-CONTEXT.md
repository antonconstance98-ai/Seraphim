# Phase 12: Context Compression - Context

**Gathered:** 2026-04-03
**Status:** Ready for planning

<domain>
## Phase Boundary

Create a universal MiniMax-powered compression utility (`minimax-compress.js`) that summarizes large inputs to save tokens. Integrates with existing hooks via auto-triggering thresholds and a manual `require()` interface. Applies to diffs, files, tool outputs, conversations — anything with large token counts.

</domain>

<decisions>
## Implementation Decisions

### Trigger strategy
- **D-01:** Threshold-based auto-compression plus manual utility. Three auto-triggers:
  1. `gsd-context-monitor.js` detects >60% context usage → compress earlier conversation context
  2. A tool output exceeds 10K tokens → compress before injecting as `additionalContext`
  3. A git diff exceeds 8K characters → compress before sending to review hooks
- **D-02:** Also exposed as `require('./minimax-compress')` for any hook to call manually. Function signature: `compress(text, { maxOutputTokens, purpose })` where purpose guides the compression style ("summarize code diff", "condense tool output", etc.).
- **D-03:** All thresholds configurable in the `minimax` settings block: `"compress_context_pct": 60`, `"compress_tool_output_tokens": 10000`, `"compress_diff_chars": 8000`.

### Hook integration points
- **D-04:** `PostToolUse` — wrap large tool outputs before they consume context. Check `tool_result` length against threshold. If over, compress and output the summary as `additionalContext` instead.
- **D-05:** `gsd-context-monitor.js` integration — when context usage hits the warning threshold, the monitor calls `minimax-compress` to summarize accumulated context and injects the compressed version.
- **D-06:** Utility callable from review hooks — `codex-review-gate.js` can compress large diffs before sending to Codex/MiniMax review, keeping review costs down.

### Compression behavior
- **D-07:** Compression prompt includes the `purpose` parameter so MiniMax knows what to preserve. For code diffs: preserve file names, function signatures, and the nature of changes. For tool outputs: preserve key data points, errors, and actionable information. For conversations: preserve decisions, blockers, and action items.
- **D-08:** Output always starts with `[Compressed from ~{N}K tokens]` header so downstream consumers know they're reading a summary, not the original.

### Claude's Discretion
- Exact compression prompts for each purpose type
- Whether to cache compressed results (avoid re-compressing the same content)
- Compression ratio targets (how aggressively to summarize)
- Whether to store originals alongside compressed versions for audit

</decisions>

<specifics>
## Specific Ideas

- The two-stage context compression pattern from the research: use M-2.7 ($0.30/Mtok) to summarize large codebases, then send only the summary to Opus — avoids Opus premium pricing on long contexts ($10/$37.50 per Mtok above 200K)
- Compression should be lossy but not destructive — key decisions, error messages, and file paths must always survive
- The `[Compressed from ~{N}K tokens]` header is important so Opus knows when it's working from a summary and can request the full content if needed

</specifics>

<canonical_refs>
## Canonical References

### Phase 8 Foundation (dependency)
- `.planning/phases/08-minimax-foundation/08-CONTEXT.md` — `runMinimax()` interface for compression calls

### Context Monitoring
- `~/.claude/hooks/gsd-context-monitor.js` — Existing context usage detection (WARNING at 35%, CRITICAL at 25% remaining). Compression integrates with this.

### Research
- `minimax-m2.7-synthesis.md` §6 "Two-Stage Context Compression Pattern" — Cost analysis showing $1.71 vs $17.50 for large codebase reasoning
- `research/deep-research-report(4).md` — "Two-stage context compression pattern" recommendation

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `gsd-context-monitor.js` — Already detects context usage thresholds and injects warnings. Can be extended to call `minimax-compress` when threshold hit.
- `minimax-exec.js` (from Phase 8) — `runMinimax(prompt, opts)` for the compression API call.

### Established Patterns
- PostToolUse hooks receive `tool_result` content that can be measured for length
- `additionalContext` injection (max 10,000 chars) — compressed output must fit within this limit
- Context bridge file at `/tmp/claude-ctx-{session_id}.json` provides usage metrics

### Integration Points
- `gsd-context-monitor.js` reads context from bridge file, currently outputs warnings. Add compression call when threshold exceeded.
- PostToolUse hook chain runs in order: `gsd-context-monitor` → `codex-token-logger` → `codex-wave-validator`. New compression logic can be added to `gsd-context-monitor` or as a separate hook.

</code_context>

<deferred>
## Deferred Ideas

- Streaming compression (compress as content arrives rather than after) — complex, evaluate if batch compression is too slow
- Compression quality metrics (how much information is lost) — hard to measure, defer to manual spot-checks

</deferred>

---

*Phase: 12-context-compression*
*Context gathered: 2026-04-03*
