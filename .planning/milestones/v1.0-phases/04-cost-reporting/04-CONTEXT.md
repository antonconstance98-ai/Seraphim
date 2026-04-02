# Phase 4: Cost Reporting - Context

**Gathered:** 2026-04-02
**Status:** Ready for planning

<domain>
## Phase Boundary

Generate human-readable session cost reports that prove the value of the multi-model integration. This phase delivers: a cost reporter script that reads `.planning/token-log.jsonl`, computes actual spend vs Opus-only baseline, and writes Markdown reports to `.planning/session-reports/YYYY-MM-DD.md`. Triggered automatically via SessionStart hook (reports on prior session) and available as a standalone script. No new review logic, no hook modifications beyond SessionStart registration.

</domain>

<decisions>
## Implementation Decisions

### Report Content & Layout
- **D-01:** Report includes three sections: summary table (total actual vs baseline, savings amount + percentage), breakdown by task type (routing, review, wave-validation, multi-round-plan-review), and model comparison table
- **D-02:** Savings displayed as both dollar amount and percentage — e.g. "Saved $4.20 (78%)"
- **D-03:** Report includes per-hook-type breakdowns grouped by task_type field from token-log.jsonl — shows where value comes from
- **D-04:** Opus-only baseline estimated by re-pricing every Codex token record using Opus rates from the existing PRICING table in codex-exec.js

### Trigger Mechanism
- **D-05:** Report triggered by SessionStart hook reading the previous session's token log — no Stop hook overhead, generates report for the prior session when a new one begins
- **D-06:** Also supports manual generation via standalone script execution — `node ~/.claude/hooks/codex-cost-reporter.js`
- **D-07:** Empty token-log.jsonl causes silent skip — no report generated if no Codex activity occurred
- **D-08:** Reports kept as one per date — `.planning/session-reports/2026-04-02.md`; append session count suffix if multiple sessions per day

### Opus Baseline Estimation
- **D-09:** Codex-only comparison — Claude Code session tokens are NOT included since they're identical regardless; report focuses purely on Codex savings
- **D-10:** Reuse `computeCost` from codex-exec.js — no separate pricing config needed

### Claude's Discretion
- Report Markdown formatting and visual layout
- Session boundary detection logic (how to identify which log entries belong to the "previous session")
- Session count suffix format for multiple same-day reports
- Any summary statistics beyond the required sections

</decisions>

<code_context>
## Existing Code Insights

### Reusable Assets
- **codex-exec.js** — `computeCost(tokens, model)` function with PRICING table for gpt-5.4, gpt-5.4-mini, claude-opus-4-6
- **token-log.jsonl** — JSONL format with model, task_type, tokens_in, tokens_out, cost, timestamp fields (written by all Phase 1-3 hooks)
- **SessionStart hook pattern** — already exists in settings.json for gsd-context-monitor.js

### Established Patterns
- All hooks follow Node.js stdin/stdout JSON pattern
- Token log entries use `task_type` field: 'routing', 'review', 'wave-validation', 'multi-round-plan-review'
- `fs.appendFileSync` for JSONL writes, `fs.readFileSync` + split for reads

### Integration Points
- New SessionStart hook entry in `~/.claude/settings.json`
- Reads `.planning/token-log.jsonl` (existing)
- Writes to `.planning/session-reports/` directory (new)
- Requires `computeCost` from `~/.claude/hooks/codex-exec.js`

</code_context>

<specifics>
## Specific Ideas

No specific requirements — open to standard approaches based on existing codebase patterns.

</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope.

</deferred>

---

*Phase: 04-cost-reporting*
*Context gathered: 2026-04-02*
