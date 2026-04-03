# Phase 7: Charts & Hook Integration - Context

**Gathered:** 2026-04-03
**Status:** Ready for planning
**Mode:** Auto-generated (final integration phase — decisions clear from ROADMAP + prior phases)

<domain>
## Phase Boundary

Chart.js time-series and comparison charts render in the dashboard, and the SessionStart hook automatically regenerates the dashboard on every session open with session delay under 2 seconds.

Requirements: CHART-01, CHART-02, CHART-03, INTG-02

</domain>

<decisions>
## Implementation Decisions

### Chart.js Charts
- Daily cost/savings line chart: X-axis = dates, two Y-series (actual cost, Opus baseline), savings shown as shaded area between
- Daily/weekly toggle: client-side JS button that regroups the timeSeries data from DASHBOARD_DATA without page reload
- Project comparison horizontal bar chart: sorted by savings %, one bar per project showing cost vs Opus baseline side by side
- Chart.js is already inlined (Phase 6 handles sidecar loading) — just add chart initialization code in the HTML template

### SessionStart Hook
- Register `codex-global-aggregator.js` in `~/.claude/settings.json` SessionStart hook array
- Place AFTER `codex-cost-reporter.js` (per-project first, then global aggregation)
- Set timeout to 30s (discovery can be slow with many projects)
- The aggregator already calls `generateDashboard()` — no additional wiring needed

### Claude's Discretion
- Chart colors and styling (use Chart.js defaults with dark theme adjustments)
- Chart canvas dimensions and responsive behavior
- Weekly grouping logic (ISO week or simple 7-day buckets)
- Exact placement of charts in the HTML page (recommended: between summary cards and per-project table)

</decisions>

<code_context>
## Existing Code Insights

### Reusable Assets
- `codex-dashboard-generator.js` — already has `computeMetrics()` returning `timeSeries` array (daily grouped) and full DASHBOARD_DATA
- `generateDashboard()` — HTML template that already inlines Chart.js source; chart canvases need to be added
- `~/.claude/dashboard/assets/chart.min.js` — Chart.js 4.5.1 already downloaded and SHA-256 verified
- `~/.claude/settings.json` — existing SessionStart hook array with `codex-cost-reporter.js`

### Established Patterns
- Chart.js initialization via `new Chart(ctx, config)` in inline `<script>` block
- Data passed via `const DASHBOARD_DATA = ...` already serialized in HTML
- Dark theme CSS variables already defined in the dashboard

### Integration Points
- Modify `codex-dashboard-generator.js` — add chart canvases and initialization JS to the HTML template
- Modify `~/.claude/settings.json` — add aggregator to SessionStart hooks array
- No new files needed

</code_context>

<specifics>
## Specific Ideas

- Charts should appear between the summary cards section and the per-project table
- Daily/weekly toggle: simple button/link pair that destroys and recreates the chart with regrouped data
- Line chart: use semi-transparent fill between the two lines to visualize savings area
- Bar chart: horizontal bars, project names as Y-axis labels, sorted by savings % descending

</specifics>

<deferred>
## Deferred Ideas

None — this is the final phase of v1.1.

</deferred>
