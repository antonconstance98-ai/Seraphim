# Phase 14: Three-Model Reporting - Context

**Gathered:** 2026-04-03
**Status:** Ready for planning

<domain>
## Phase Boundary

Update the entire cost tracking and dashboard pipeline to support three models (Opus, Codex, MiniMax). Includes token logging, session cost reports, global aggregator, and HTML dashboard with three-model charts and a fallback events panel.

</domain>

<decisions>
## Implementation Decisions

### Dashboard layout
- **D-01:** Combined charts with a third color/series for MiniMax. Existing time-series chart gets a MiniMax line. Cost breakdown pie chart shows Opus vs Codex vs MiniMax split.
- **D-02:** New "Fallback Events" panel showing when Codex→MiniMax fallbacks occurred, why (rate limit, quota, timeout), and how much each fallback cost. This is a new dashboard section, not a modification to existing charts.
- **D-03:** Savings calculation uses corrected Opus 4.6 baseline ($5/$25, not the old $15/$75). All savings percentages reflect the true Opus 4.6 cost.

### Token logging updates
- **D-04:** `codex-token-logger.js` already handles different models via the `model` field. Ensure it correctly recognizes `minimax-m2.7` entries (from Phase 11 bug scanner, Phase 9 dual review, Phase 10 adversarial review, Phase 12 compression, Phase 13 execution fallback).
- **D-05:** New log fields for v2.0 entries: `dual_review: true/false` (Phase 9), `review_round: 1|2` + `round_model` (Phase 10), `fallback_from: 'codex'` (Phase 13), `compression: true` (Phase 12). Backward compatible — old entries without these fields still parse fine.

### Cost reporter updates
- **D-06:** `codex-cost-reporter.js` SessionStart report shows three-model breakdown: Opus orchestration cost, Codex execution cost, MiniMax analysis cost, total, and savings vs Opus-only baseline.
- **D-07:** Report includes fallback event count and cost: "Codex→MiniMax fallbacks: N events, $X.XX additional cost."

### Global aggregator updates
- **D-08:** `codex-global-aggregator.js` already aggregates token-log.jsonl across projects. No discovery changes needed — MiniMax entries are in the same JSONL files. Pricing computation updated via the corrected `codex-pricing.js` from Phase 8.

### Dashboard generator updates
- **D-09:** `codex-dashboard-generator.js` adds MiniMax as third series in Chart.js charts. Color scheme: Opus = existing color, Codex = existing color, MiniMax = new distinct color (e.g., teal or green — Claude's discretion).
- **D-10:** Fallback events panel rendered as a table: date, reason, tokens consumed, cost, source project. Sorted by most recent first.

### Claude's Discretion
- Exact color for MiniMax in charts
- Fallback events panel position on dashboard (top, bottom, sidebar)
- Whether to add a "Model Efficiency" metric (quality per dollar)
- Summary statistics format in SessionStart report

</decisions>

<specifics>
## Specific Ideas

- The dashboard should make it immediately obvious how much the three-model setup saves vs running everything on Opus — this is the success metric for the entire v2.0 milestone
- Fallback events are a health indicator — too many fallbacks means the Codex subscription limits are being hit too often and a Token Plan for MiniMax might be worthwhile
- Historical data should show the transition from two-model to three-model clearly (before/after v2.0)

</specifics>

<canonical_refs>
## Canonical References

### All Prior Phases (dependencies)
- `.planning/phases/08-minimax-foundation/08-CONTEXT.md` — Corrected pricing in `codex-pricing.js`
- `.planning/phases/09-dual-review-gate/09-CONTEXT.md` — `dual_review` log field (D-07)
- `.planning/phases/10-adversarial-plan-review/10-CONTEXT.md` — `round_model` log field (D-07)
- `.planning/phases/11-posttooluse-bug-scanner/11-CONTEXT.md` — `task_type: 'post-scan'` log entries
- `.planning/phases/12-context-compression/12-CONTEXT.md` — `compression: true` log field
- `.planning/phases/13-codex-execution-pipeline/13-CONTEXT.md` — `source: 'cli-fallback'/'api-fallback'` entries

### Existing Reporting Infrastructure
- `~/.claude/hooks/codex-token-logger.js` — Token logging hook to update
- `~/.claude/hooks/codex-cost-reporter.js` — SessionStart cost report to update
- `~/.claude/hooks/codex-global-aggregator.js` — Global JSONL aggregator to update
- `~/.claude/hooks/codex-dashboard-generator.js` — HTML dashboard generator to update
- `~/.claude/hooks/codex-pricing.js` — Pricing module (already updated in Phase 8)
- `~/.claude/dashboard/dashboard.html` — Current dashboard output

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `codex-dashboard-generator.js` — Full Chart.js dashboard generator. Already handles multiple data series. Adding a third color/series is incremental.
- `codex-cost-reporter.js` — Reads `token-log.jsonl`, groups by model, computes savings. Extending to three models is straightforward.
- `codex-global-aggregator.js` — Discovery + merge pipeline. No structural changes needed — MiniMax entries are in the same JSONL files.

### Established Patterns
- Dashboard uses inline Chart.js (sidecar file with SHA-256 integrity)
- Atomic write-then-rename for dashboard.html
- JSONL schema: `{ timestamp, session_id, model, source, task_type, tokens, cost_usd }`
- Pricing computed at report time (not at log time) — so pricing corrections apply retroactively to historical data

### Integration Points
- `codex-pricing.js` — Already updated with MiniMax pricing in Phase 8. All cost computation flows through this module.
- `codex-dashboard-generator.js` — Receives aggregated data from `codex-global-aggregator.js` and renders HTML. Add MiniMax series to existing chart configuration.
- SessionStart hook chain: `codex-cost-reporter` → `codex-global-aggregator` (which calls `codex-dashboard-generator`). No new hooks needed — just module updates.

</code_context>

<deferred>
## Deferred Ideas

- Per-model quality tracking (pass rate, retry rate) alongside cost metrics — requires structured verdict logging beyond current ALLOW/BLOCK
- Export dashboard data as JSON API for external consumption
- Email/Slack notification when daily spend approaches $15 limit

</deferred>

---

*Phase: 14-three-model-reporting*
*Context gathered: 2026-04-03*
