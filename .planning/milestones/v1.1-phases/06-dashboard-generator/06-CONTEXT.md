# Phase 6: Dashboard Generator - Context

**Gathered:** 2026-04-03
**Status:** Ready for planning

<domain>
## Phase Boundary

The dashboard generator reads `global.jsonl` and produces a `dashboard.html` that opens from `file://` in a browser with all tables, session drill-down, and issue log rendering correctly, written via atomic rename so concurrent sessions cannot corrupt it.

Requirements: DASH-01, DASH-02, DASH-03, DASH-04, DASH-05, DASH-06, DASH-07, SESS-01, SESS-02, INTG-01, INTG-03, INTG-04, INTG-05

</domain>

<decisions>
## Implementation Decisions

### Dashboard Layout & Visual Design
- Dark theme (dark background, light text) — matches terminal-first workflow, easier on eyes for metrics dashboards
- Single-page vertical scroll with section headers — simple, no routing needed, works with file://
- Large number cards at top for global summary (total cost, savings %, calls, reviews) — dashboard convention, instant overview
- Zebra-striped rows with hover highlight for all tables — readable at a glance

### Session & Data Interaction
- Click row to expand inline accordion for session drill-down — shows call breakdown below the row, no page navigation needed
- Last 50 sessions shown, paginated 20 per page — balanced history depth vs page size
- BLOCK issue log in reverse-chronological list with timestamp, project, summary — most recent issues first
- Unattributed calls shown as separate row at bottom of per-project table labeled "Unattributed" with italic styling

### Technical Approach
- Chart.js downloaded once to `~/.claude/dashboard/assets/chart.min.js`, inlined into HTML at generation time — works offline
- Atomic write via `.dashboard.html.tmp` then `fs.renameSync` — prevents concurrent session corruption
- Data serialized as `const DASHBOARD_DATA = ${JSON.stringify(data)}` inline in `<script>` — no fetch needed from file://
- Inline `<style>` block with CSS custom properties for theming — self-contained, no external deps

### Claude's Discretion
- Exact color values for dark theme (use standard dark theme conventions)
- Specific CSS layout details (padding, margins, font sizes)
- Section ordering on the page (recommended: summary cards → project table → model split → task type → session history → BLOCK log → footer)
- Pagination UI pattern for session history

</decisions>

<code_context>
## Existing Code Insights

### Reusable Assets
- `codex-global-aggregator.js` — exports `aggregate()` function; writes `global.jsonl` with enriched records (project_name, project_path, opus_baseline_usd)
- `codex-pricing.js` — exports `computeOpusCost()`, `OPUS_PRICING`, `CODEX_PRICING` for any additional cost calculations
- `codex-cost-reporter.js` — established pattern for `generateReport()` function producing structured output from JSONL records

### Established Patterns
- All hook scripts use stdin JSON for event data, stdout JSON for additionalContext
- Silent fail: outer try/catch with process.exit(0) — never block session
- Template literal string concatenation for HTML/Markdown generation (used in codex-cost-reporter.js)
- Write-then-rename for atomic file writes (used in codex-global-aggregator.js)

### Integration Points
- New script at `~/.claude/hooks/codex-dashboard-generator.js`
- Called by aggregator via `require('./codex-dashboard-generator').generateDashboard(dashboardDir)`
- Reads from: `~/.claude/dashboard/global.jsonl`
- Writes to: `~/.claude/dashboard/dashboard.html`
- Chart.js sidecar at: `~/.claude/dashboard/assets/chart.min.js`

</code_context>

<specifics>
## Specific Ideas

- Section order: summary cards → per-project table → model split → task type distribution → session history → BLOCK log → footer
- Summary cards should show: Total Cost, Opus Baseline, Savings %, Total Calls, Total Reviews, Catch Rate
- Per-project table columns: Project, Calls, Cost, Opus Baseline, Savings %, Catch Rate, Cache Efficiency
- Model split: simple two-column comparison (GPT-5.4 vs GPT-5.4-mini)

</specifics>

<deferred>
## Deferred Ideas

- Date-range filter for all views (v2)
- Cost anomaly detection (v2)
- Project comparison bar chart (Phase 7 — CHART-03)
- Daily/weekly time trend charts (Phase 7 — CHART-01, CHART-02)

</deferred>
