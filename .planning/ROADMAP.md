# Roadmap: Claude X Codex

## Milestones

- ✅ **v1.0 Claude X Codex** — Phases 1-4 (shipped 2026-04-02) — [archive](milestones/v1.0-ROADMAP.md)
- 🚧 **v1.1 Global Metrics Dashboard** — Phases 5-7 (in progress)

## Phases

<details>
<summary>✅ v1.0 Claude X Codex (Phases 1-4) — SHIPPED 2026-04-02</summary>

- [x] Phase 1: Foundation (3/3 plans) — completed 2026-04-02
- [x] Phase 2: Review Gate & GSD Integration (2/2 plans) — completed 2026-04-02
- [x] Phase 3: Plan Review Loop & Superpowers (2/2 plans) — completed 2026-04-02
- [x] Phase 4: Cost Reporting (1/1 plan) — completed 2026-04-02

</details>

### 🚧 v1.1 Global Metrics Dashboard (In Progress)

**Milestone Goal:** A self-contained HTML dashboard at `~/.claude/dashboard/` showing Codex usage, costs, savings, and review activity across every project on this machine — auto-updated on every session start.

- [ ] **Phase 5: Data Pipeline** - Global aggregator that discovers, deduplicates, and incrementally merges all per-project token logs into a single verified store
- [ ] **Phase 6: Dashboard Generator** - HTML generator that transforms the global log into a self-contained dashboard with tables, session history, and issue log
- [ ] **Phase 7: Charts & Hook Integration** - Chart.js visualizations and SessionStart hook wiring so the dashboard auto-regenerates on every session open

## Phase Details

### Phase 5: Data Pipeline
**Goal**: The global aggregator runs standalone, correctly merges all per-project token logs into `global.jsonl`, handles null session IDs as "unattributed", and completes a repeat run in under 5 ms
**Depends on**: Phase 4
**Requirements**: PIPE-01, PIPE-02, PIPE-03, PIPE-04
**Success Criteria** (what must be TRUE):
  1. Running `node ~/.claude/hooks/codex-global-aggregator.js` produces a `global.jsonl` containing records from all known projects with no duplicate entries
  2. Records with `session_id: null` appear in `global.jsonl` tagged as "Unattributed" rather than being missing
  3. A second consecutive run with no new data completes in under 5 ms and adds zero new records to `global.jsonl`
  4. Discovery roots and depth limits are read from `~/.claude/dashboard/config.json` with sensible defaults so no projects are silently skipped
**Plans:** 3 plans
Plans:
- [x] 05-01-PLAN.md — Centralized pricing module and refactor existing hooks to use it
- [x] 05-02-PLAN.md — Global aggregator with discovery, dedup, incremental reads, and null session handling
- [x] 05-03-PLAN.md — Mtime-gated discovery cache to bring no-op run time under 5 ms (gap closure)

### Phase 6: Dashboard Generator
**Goal**: The dashboard generator reads `global.jsonl` and produces a `dashboard.html` that opens from `file://` in a browser with all tables and the issue log rendering correctly, written via atomic rename so concurrent sessions cannot corrupt it
**Depends on**: Phase 5
**Requirements**: DASH-01, DASH-02, DASH-03, DASH-04, DASH-05, DASH-06, DASH-07, SESS-01, SESS-02, INTG-01, INTG-03, INTG-04, INTG-05
**Success Criteria** (what must be TRUE):
  1. Opening `~/.claude/dashboard/dashboard.html` from `file://` in a browser shows global summary cards, per-project breakdown table, model split section, review BLOCK log, cache efficiency column, unattributed calls row, and task type distribution
  2. Clicking a session row in the session history table expands to show individual call breakdown for that session
  3. The dashboard footer shows the ISO timestamp of when it was last generated
  4. Starting two Claude Code sessions simultaneously leaves `dashboard.html` intact and fully readable (no torn or empty file)
**Plans:** 2/2 plans executed
Plans:
- [x] 06-01-PLAN.md — Data processing module (computeMetrics) and Chart.js sidecar setup
- [x] 06-02-PLAN.md — HTML template rendering, session drill-down, aggregator wiring, and visual verification

### Phase 7: Charts & Hook Integration
**Goal**: Chart.js time-series and comparison charts render in the dashboard, and the SessionStart hook automatically regenerates the dashboard on every session open with session delay under 2 seconds
**Depends on**: Phase 6
**Requirements**: CHART-01, CHART-02, CHART-03, INTG-02
**Success Criteria** (what must be TRUE):
  1. The dashboard shows a daily cost/savings line chart and a project comparison horizontal bar chart, both rendering from `file://` without browser security errors
  2. Clicking the daily/weekly toggle regroups the line chart data client-side without a page reload
  3. Opening a Claude Code session automatically updates `global.jsonl` and regenerates `dashboard.html` with the new session's data visible in the dashboard
  4. Session open delay attributable to the hook is under 2 seconds on a cold scan across all known projects
**Plans:** 1/2 plans executed
Plans:
- [ ] 07-01-PLAN.md — Chart.js line chart (daily cost/savings), weekly toggle, and project bar chart
- [x] 07-02-PLAN.md — Register codex-global-aggregator.js as SessionStart hook in settings.json

## Progress

| Phase | Milestone | Plans Complete | Status | Completed |
|-------|-----------|----------------|--------|-----------|
| 1. Foundation | v1.0 | 3/3 | Complete | 2026-04-02 |
| 2. Review Gate & GSD Integration | v1.0 | 2/2 | Complete | 2026-04-02 |
| 3. Plan Review Loop & Superpowers | v1.0 | 2/2 | Complete | 2026-04-02 |
| 4. Cost Reporting | v1.0 | 1/1 | Complete | 2026-04-02 |
| 5. Data Pipeline | v1.1 | 3/3 | Complete |  |
| 6. Dashboard Generator | v1.1 | 2/2 | Complete |  |
| 7. Charts & Hook Integration | v1.1 | 1/2 | In Progress|  |
