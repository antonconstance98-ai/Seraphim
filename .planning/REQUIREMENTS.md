# Requirements: Claude X Codex

**Defined:** 2026-04-02
**Core Value:** Every task goes to the model that's best at it — Opus for reasoning and architecture, Codex for fast execution — with cross-model review catching what either model misses alone.

## v1.1 Requirements

Requirements for Global Metrics Dashboard milestone. Each maps to roadmap phases.

### Data Pipeline

- [ ] **PIPE-01**: Global aggregator scans all projects for `token-log.jsonl` and merges records into `~/.claude/dashboard/global.jsonl` with deduplication
- [x] **PIPE-02**: Discovery roots are configurable via `~/.claude/dashboard/config.json` with sensible defaults covering all CLAUDE.md key paths
- [ ] **PIPE-03**: Aggregator uses mtime-gated incremental reads to skip unchanged files
- [ ] **PIPE-04**: Records with null `session_id` are tracked as "unattributed" category rather than silently dropped

### Dashboard Display

- [ ] **DASH-01**: Dashboard shows global summary cards (total cost, Opus baseline, savings %, total calls, total reviews)
- [ ] **DASH-02**: Per-project breakdown table shows project name, calls, actual cost, Opus baseline, savings %, catch rate
- [ ] **DASH-03**: Model split section shows GPT-5.4 vs GPT-5.4-mini call counts, tokens, and costs
- [ ] **DASH-04**: Review activity section shows global catch rate with BLOCK issue log (timestamp, project, issue summary)
- [ ] **DASH-05**: Cache efficiency metric shows cached vs uncached input token ratio per project
- [ ] **DASH-06**: Unattributed calls section surfaces null-session records with cost attribution
- [ ] **DASH-07**: Task type distribution shows what Codex is used for (review, implementation, bulk-ops, etc.)

### Charts & Trends

- [ ] **CHART-01**: Daily cost/savings line chart using Chart.js
- [ ] **CHART-02**: Daily/weekly toggle switches grouping client-side
- [ ] **CHART-03**: Project comparison horizontal bar chart (sorted by savings %)

### Session Tracking

- [ ] **SESS-01**: Session history table lists recent sessions across all projects (date, project, calls, cost, savings, catch rate)
- [ ] **SESS-02**: Per-session drill-down shows individual call breakdown when a session row is clicked

### Integration

- [ ] **INTG-01**: Self-contained HTML dashboard at `~/.claude/dashboard/dashboard.html` (inline CSS/JS, opens from `file://`)
- [ ] **INTG-02**: SessionStart hook auto-regenerates the dashboard on every session
- [ ] **INTG-03**: Dashboard shows last-updated timestamp in footer
- [ ] **INTG-04**: Dashboard writes use atomic write-then-rename for concurrent session safety
- [ ] **INTG-05**: Chart.js stored as sidecar file at `~/.claude/dashboard/assets/chart.min.js` and inlined at generation time

## v2 Requirements

Deferred to future release. Tracked but not in current roadmap.

### Advanced Analytics

- **ADV-01**: Date-range filter for all dashboard views
- **ADV-02**: Cost anomaly detection (spike alerts in SessionStart context)
- **ADV-03**: Token efficiency metric (output/input ratio trends)

## Out of Scope

| Feature | Reason |
|---------|--------|
| Web server / hosted dashboard | Contradicts terminal-first, local-only design; file:// is simpler |
| Real-time auto-refresh | JSONL is append-only; regeneration on SessionStart is sufficient |
| SQLite or any database | JSONL aggregation is fast enough at this data volume; no second source of truth |
| Cross-machine aggregation | Requires network, auth, server; explicitly out of scope per PROJECT.md |
| Plotly.js charts | 3-4x larger bundle than Chart.js; overkill for line + bar charts |
| Interactive chart editing | High JS complexity for a read-only metrics view |

## Traceability

Which phases cover which requirements. Updated during roadmap creation.

| Requirement | Phase | Status |
|-------------|-------|--------|
| PIPE-01 | Phase 5 | Pending |
| PIPE-02 | Phase 5 | Complete |
| PIPE-03 | Phase 5 | Pending |
| PIPE-04 | Phase 5 | Pending |
| DASH-01 | Phase 6 | Pending |
| DASH-02 | Phase 6 | Pending |
| DASH-03 | Phase 6 | Pending |
| DASH-04 | Phase 6 | Pending |
| DASH-05 | Phase 6 | Pending |
| DASH-06 | Phase 6 | Pending |
| DASH-07 | Phase 6 | Pending |
| SESS-01 | Phase 6 | Pending |
| SESS-02 | Phase 6 | Pending |
| INTG-01 | Phase 6 | Pending |
| INTG-03 | Phase 6 | Pending |
| INTG-04 | Phase 6 | Pending |
| INTG-05 | Phase 6 | Pending |
| CHART-01 | Phase 7 | Pending |
| CHART-02 | Phase 7 | Pending |
| CHART-03 | Phase 7 | Pending |
| INTG-02 | Phase 7 | Pending |

**Coverage:**
- v1.1 requirements: 21 total
- Mapped to phases: 21
- Unmapped: 0

---
*Requirements defined: 2026-04-02*
*Last updated: 2026-04-02 after roadmap creation — all 21 requirements mapped*
