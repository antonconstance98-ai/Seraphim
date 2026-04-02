# Feature Research

**Domain:** Local HTML metrics dashboard — multi-project AI model usage, costs, savings, and review quality
**Researched:** 2026-04-02
**Confidence:** HIGH

> **Milestone scope:** v1.1 Global Metrics Dashboard only. Features from v1.0 (hook scripts,
> routing, review gate) are already shipped and not re-researched here. This file covers only
> what needs to be built for the dashboard.

---

## Existing Data Contract (token-log.jsonl Schema)

All features depend on this schema. Verified consistent across both live projects on this machine.

```jsonc
{
  "timestamp":        "2026-04-02T22:52:30.644Z",   // ISO 8601 — required for time trends
  "session_id":       "30092b41-...",                // nullable string; groups records into sessions
  "model":            "gpt-5.4",                     // "gpt-5.4" | "gpt-5.4-mini"
  "source":           "cli",                         // "cli" | "api"
  "task_type":        "review",                      // "review" | "multi-round-plan-review" | ...
  "review_task_type": "feature",                     // nullable; "feature" | "plan" | ...
  "verdict":          "ALLOW",                       // nullable; "ALLOW" | "BLOCK"
  "block_summary":    "...",                         // nullable string — issue Codex caught
  "tokens": {
    "input":            12860,
    "cached_input":     10624,
    "output":           365,
    "reasoning_output": 0
  },
  "cost_usd":         0.04908,                       // actual cost already computed per record
  "rate_limit_pct":   null                           // nullable float 0–100
}
```

**Critical schema gaps for aggregation (must be inferred at read time, not stored):**

| Field Needed | Not In Schema | How to Derive |
|---|---|---|
| Project name | Not stored | Infer from file path: last path segment before `.planning/` |
| Opus baseline cost | Not stored | Compute at read time: reprice token volumes at Opus rates ($15/$3.75/$75 per 1M in/cached/out) |
| Savings amount | Not stored | `opusBaseline - cost_usd` per record |
| Session date | Not stored directly | `timestamp.slice(0, 10)` → YYYY-MM-DD |

**Known projects with token-log.jsonl (live on this machine):**
- `/home/alucard/projects/Claude_X_Codex/.planning/token-log.jsonl`
- `/home/alucard/projects/The-Crucible/.planning/token-log.jsonl`

---

## Feature Landscape

### Table Stakes (Users Expect These)

Features required for the dashboard to answer more questions than the existing Markdown session
reports. Missing any of these means the dashboard is a regression, not an upgrade.

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| Global cost summary | Primary reason to build this — one number across all projects | LOW | Sum `cost_usd` + compute Opus baseline across all JSONL files |
| Total savings % vs Opus baseline | The core success metric from v1.0 (86.7% demonstrated) — must be at the top | LOW | `(opusBaseline - actualCost) / opusBaseline * 100` |
| Per-project breakdown table | User needs to know which project drove which cost | LOW | Group by inferred project name; columns: project, calls, actual cost, savings, catch rate |
| Review catch-rate (global + per-project) | Cross-project quality metric; without it, "how well is Codex working?" is unanswerable | LOW | `reviewBlock / reviewTotal * 100`; filter where `task_type === 'review'` |
| Model split (GPT-5.4 vs GPT-5.4-mini) | Two models billed at different rates; user must know the breakdown | LOW | Group by `model` field |
| Session history list | Session-level granularity that the per-day reports obscure | MEDIUM | Group by `session_id` (non-null); sort by most recent; columns: project, date, calls, cost, catch rate |
| BLOCK issue log | Shows exactly what Codex caught — the "proof of value" log | LOW | Filter where `verdict === 'BLOCK'` and `block_summary !== null` |
| Time trend chart | Makes cost trajectory visible — impossible to see in static Markdown tables | MEDIUM | Aggregate `cost_usd` by YYYY-MM-DD; line chart; one series per project or stacked total |
| Self-contained HTML (no server) | Opened with `file://` in a browser; no Node/Python server; no `fetch()` to local files | MEDIUM | Inline all CSS + Chart.js source; serialize all data as `const DASHBOARD_DATA = {...}` in a `<script>` block at generation time |
| Auto-regenerate on SessionStart | Dashboard is useless if it goes stale; must stay current automatically | LOW | Add a second SessionStart hook (`codex-dashboard-gen.js`) alongside existing `codex-cost-reporter.js` |
| Last-updated timestamp | Tells user when data was last refreshed; prevents acting on stale data | LOW | Write `new Date().toISOString()` into the HTML footer at generation time |

### Differentiators (What Makes This More Valuable Than Markdown Reports)

These are the features that justify building an HTML dashboard at all, rather than continuing with
per-project Markdown session reports.

| Feature | Value Proposition | Complexity | Notes |
|---------|-------------------|------------|-------|
| Time trend line chart with daily/weekly toggle | Surfaces cost trajectory and savings trend over time — impossible in tabular session reports; weekly view removes daily noise for leadership-level reading | MEDIUM | Daily grouping is native (`timestamp.slice(0,10)`); weekly toggle regroups daily data client-side into ISO week buckets (JS: `getFullYear() + '-W' + getWeek()`) |
| Project comparison bar chart | Side-by-side cost, savings, and catch rate across all projects at a glance | MEDIUM | Horizontal bar chart; one bar per project sorted by savings %; Chart.js bar type |
| Cache efficiency per project | Shows what fraction of Codex input tokens hit the cache (cost avoidance within Codex itself) | LOW | `cached_input / (input + cached_input) * 100` per project; shown as a column in the breakdown table |
| Session history with per-session stats | Drill into individual sessions across all projects — what was done, what was caught, what it cost | MEDIUM | Group non-null `session_id` records; compute per-session cost, call count, catch rate; link to project |
| Orphan call tracking | Records with `session_id === null` (hook calls outside a Claude session) — surfacing these explains cost attribution gaps | LOW | Bucket null-session records into a synthetic "orphan" group per project |

### Anti-Features (Do Not Build)

| Feature | Why It Seems Good | Why Problematic | What to Do Instead |
|---------|-------------------|-----------------|-------------------|
| Real-time auto-refresh | Dashboard feels "live" | JSONL files are append-only and written by hooks, not streamed. Polling them with `setInterval` adds complexity, risks reading a partially-written line, and provides zero value for a tool that regenerates on SessionStart. | Regenerate at SessionStart; show "Last updated" timestamp. User knows to re-open after a session. |
| Server process (Express, Python HTTP) | Enables `fetch()` for live data reloads | Violates the "no server" project requirement. Every server is a process to manage, a port to expose, and a dependency to break. The `file://` constraint eliminates this class of complexity entirely. | Inline all data as a JS variable in the generated HTML file at build time. |
| SQLite or any database | Enables complex queries over historical data | JSONL is already the canonical storage format (written by hooks, read by the reporter). Introducing a DB means a migration job, a sync job, and a second source of truth. At this data volume (hundreds of records per project), reading and aggregating JSONL at generation time is under 100 ms. | Aggregate JSONL at HTML generation time in Node.js. No DB needed. |
| Editable charts / drill-down interactivity | Feels powerful and modern | High JS complexity for a read-only metrics view. Interactive filters add 3-5x the client-side code of static charts. Research confirms dashboards with too many interactive elements increase time-to-insight by 35%. | Static charts with Chart.js tooltips (built-in) are sufficient. Add date-range filter only in v2 if users request it. |
| Web UI dashboard vs local HTML | Accessible from any device | Contradicts the project's terminal-first, local-only design. The existing `file://` constraint is a feature (no auth, no hosting, no maintenance). A web UI would require hosting, auth, and a running server. | Self-contained `~/.claude/dashboard/index.html` opened directly in browser. |
| Alert / notification system | Notify on cost spikes | Disproportionate complexity for a single-user local tool. The SessionStart hook already injects a cost summary into Claude's context (the `additionalContext` mechanism). | SessionStart hook `additionalContext` is already the notification channel. |
| Cross-machine aggregation | Show data from all machines the user works on | Requires network, auth, and a server. Explicitly out of scope per PROJECT.md. | Single-machine only; all JSONL files are local. |
| Plotly.js as the chart library | More chart types, scientific charts | Bundle is 3-4x larger than Chart.js (~1 MB minified vs ~265 KB). Requires React internally. For line + bar + pie, Chart.js is sufficient and lighter. | Use Chart.js. Register only the chart types needed to minimize inline bundle size. |

---

## Feature Dependencies

```
[Global aggregator: scan all token-log.jsonl files]
    required-by --> [Global cost summary card]
    required-by --> [Per-project breakdown table]
    required-by --> [Time trend chart]
    required-by --> [Session history table]
    required-by --> [Project comparison bar chart]
    required-by --> [BLOCK issue log]
    required-by --> [Cache efficiency metric]

[Global aggregator]
    feeds --> [Data inlined as JS variable in HTML]
                  feeds --> [All chart rendering code]

[Data inlined as JS variable]
    required-by --> [Self-contained HTML (no server, no fetch())]

[Self-contained HTML generation script (Node.js)]
    required-by --> [All rendered features]
    triggered-by --> [SessionStart hook integration]

[Time trend chart (daily)]
    enhanced-by --> [Weekly toggle] (client-side JS regroups same data)

[Session history table]
    depends-on --> [session_id grouping logic in aggregator]
```

### Dependency Notes

- **Global aggregator is the root:** Every visual feature depends on one Node.js function that
  finds all `token-log.jsonl` files and merges them with inferred project names. This must be built
  first and tested independently before any rendering code is written.

- **Project name requires path inference:** The JSONL schema stores no `project` field. The
  aggregator derives it with:
  `path.basename(path.dirname(path.dirname(logPath)))` where `logPath` ends in `.planning/token-log.jsonl`.

- **Scan paths (both must be checked):**
  - `~/projects/*/. planning/token-log.jsonl`
  - `~/.claude/projects/*/.planning/token-log.jsonl`
  Future projects may live in either location.

- **Self-contained HTML requires data inlining at generation time:** Because `file://` pages
  cannot `fetch()` local files, the generator serializes all aggregated data as a JS variable before
  writing the HTML file:
  ```html
  <script>const DASHBOARD_DATA = /* JSON.stringify(aggregated) */;</script>
  ```
  This means the HTML file is completely regenerated on each SessionStart, not incrementally updated.

- **SessionStart hook is a second hook, not a modification of the existing one:**
  `codex-cost-reporter.js` generates per-project Markdown reports. The new
  `codex-dashboard-gen.js` generates the global HTML dashboard. They run independently.
  Both are registered under the `SessionStart` hook event in `~/.claude/settings.json`.

- **Null session_id records ("orphan" calls):** Records where `session_id === null` are hook
  invocations that fired outside a Claude Code session context. They must be included in all cost
  totals but grouped separately in the session history view (not mixed with named sessions).

---

## MVP Definition

### Launch With (v1.1 — this milestone)

Minimum viable dashboard that provides visible value beyond existing per-project Markdown reports.

- [ ] `codex-dashboard-gen.js` — global aggregator + HTML generator
- [ ] `~/.claude/dashboard/index.html` — self-contained HTML with inline CSS + Chart.js
- [ ] Global summary section: total cost, Opus baseline, savings %, total calls, global catch rate
- [ ] Per-project breakdown table: project, calls, actual cost, savings $, savings %, catch rate, cache efficiency %
- [ ] Time trend line chart: daily total cost for last 30 days (stacked by project or multi-series)
- [ ] Session history table: project, session date, calls, cost, catch rate (sorted by most recent)
- [ ] BLOCK issue log: timestamp, project, task type, issue summary (most recent 20)
- [ ] "Last updated" footer with ISO timestamp
- [ ] SessionStart hook registration in `~/.claude/settings.json`
- [ ] Silent fail: any error in generator exits 0 and never blocks session start

### Add After Validation (v1.x)

Add once core dashboard is running and the data model is confirmed stable across multiple sessions.

- [ ] Weekly trend toggle — client-side JS; regroups same daily data into ISO week buckets
- [ ] Project comparison horizontal bar chart — cost and savings side-by-side per project
- [ ] Model split chart (pie or bar) — GPT-5.4 vs GPT-5.4-mini token and cost share
- [ ] Task-type breakdown column in per-project table

### Future Consideration (v2+)

Defer until the tool has been used for a full month and patterns are confirmed.

- [ ] Date range filter (client-side JS) — last 7 / 30 / 90 days selector
- [ ] Task-type drill-down — click a project row to see task breakdown in a modal or expanded section
- [ ] Export to CSV — download aggregated data for external analysis

---

## Feature Prioritization Matrix

| Feature | User Value | Implementation Cost | Priority |
|---------|------------|---------------------|----------|
| Global aggregator script | HIGH | LOW | P1 |
| Global cost summary card | HIGH | LOW | P1 |
| Per-project breakdown table | HIGH | LOW | P1 |
| BLOCK issue log | HIGH | LOW | P1 |
| Time trend line chart (daily) | HIGH | MEDIUM | P1 |
| Session history table | HIGH | MEDIUM | P1 |
| Self-contained HTML generation | HIGH | MEDIUM | P1 |
| SessionStart hook integration | HIGH | LOW | P1 |
| Cache efficiency column | MEDIUM | LOW | P2 |
| Weekly trend toggle | MEDIUM | LOW | P2 |
| Project comparison bar chart | MEDIUM | MEDIUM | P2 |
| Model split chart | MEDIUM | LOW | P2 |
| Date range filter | MEDIUM | MEDIUM | P3 |
| Task-type drill-down | LOW | HIGH | P3 |
| Export to CSV | LOW | LOW | P3 |

**Priority key:**
- P1: Must have for v1.1 launch
- P2: Should have, add in a follow-on patch
- P3: Nice to have, defer to v2+

---

## Competitor Comparison (for context)

This is a personal local tool — no direct commercial competitors. Closest analogues:

| Feature | ccusage (CLI) | Claude-Code-Usage-Monitor (CLI) | This HTML Dashboard |
|---------|--------------|----------------------------------|---------------------|
| Multi-project aggregation | Yes (`--project` flag) | No | Yes (automatic scan) |
| Time trend charts | No (tables only) | Real-time CLI bar | Yes (line chart) |
| HTML / browser output | No | No | Yes (primary output) |
| Codex (OpenAI) cost tracking | No (Claude only) | No | Yes (primary focus) |
| Cross-model savings vs Opus baseline | No | No | Yes (core metric) |
| Review catch-rate tracking | No | No | Yes |
| BLOCK issue log | No | No | Yes |
| Offline / no server | Yes | Yes | Yes |

**Gap being filled:** ccusage covers Claude Code (Anthropic) usage but has no concept of Codex
(OpenAI) costs, cross-model savings comparison, or review quality metrics. Neither tool produces
HTML output. This dashboard is the only view that ties Codex routing decisions to cost outcomes
and shows whether the v1.0 integration is delivering its claimed 86.7% savings consistently.

---

## Sources

- ccusage feature set: https://ccusage.com/ and https://github.com/ryoppippi/ccusage (verified 2026-04-02)
- Dashboard design: 12-KPI engagement threshold from https://improvado.io/blog/dashboard-design-guide (2026)
- Dashboard overload: 35% increased time-to-insight from https://www.smashingmagazine.com/2025/09/ux-strategies-real-time-dashboards/
- Time aggregation UX: https://writesonic.com/blog/introducing-aggregated-views
- Chart type selection (line for trends, bar for comparison): https://www.datacamp.com/tutorial/dashboard-design-tutorial
- Chart.js bundle size (~265 KB) and CDN: https://www.chartjs.org/docs/latest/getting-started/installation.html (verified 2026-04-02)
- Chart.js vs Plotly.js: https://www.luzmo.com/blog/plotly-js (2025)
- Existing token-log.jsonl schema: verified from live files — `/home/alucard/projects/Claude_X_Codex/.planning/token-log.jsonl` and `/home/alucard/projects/The-Crucible/.planning/token-log.jsonl`
- Existing cost reporter logic: `/home/alucard/.claude/hooks/codex-cost-reporter.js` (canonical source for aggregation patterns)
- Project requirements: `/home/alucard/projects/Claude_X_Codex/.planning/PROJECT.md`

---

*Feature research for: Local HTML metrics dashboard — Claude X Codex v1.1 milestone*
*Researched: 2026-04-02*
