# Phase 6: Adaptive Intelligence - Context

**Gathered:** 2026-04-04
**Status:** Ready for planning

<domain>
## Phase Boundary

Deliver pattern analysis engine, auto-recommendation system with human-approval guardrail, and a new Seraphim-branded dashboard with per-phase model performance heatmap, profile cost/quality comparison, and recommendation log panels.

</domain>

<decisions>
## Implementation Decisions

### Recommendation Trigger
- **D-01:** Always analyze from the very first pipeline run. Never wait for a threshold.
- **D-02:** Recommendations labeled with statistical confidence: LOW / MEDIUM / HIGH based on sample size and signal strength (e.g., p-value or effect size).
- **D-03:** Never auto-apply ANY recommendation regardless of confidence level. All changes surface to the user for explicit approval. Low-confidence findings are informational only.
- **D-04:** Rejected recommendations are logged with timestamp and reason for audit trail.

### Dashboard Integration
- **D-05:** New Seraphim-branded dashboard — separate from existing `~/.claude/dashboard/dashboard.html`. Own path, own identity, can evolve independently.
- **D-06:** Dashboard location TBD by Claude (could be `~/.seraphim/dashboard/` or `~/.claude/plugins/seraphim/dashboard/`).

### Analysis Frequency
- **D-07:** Analysis runs automatically after every complete pipeline run (triggered after Crucible phase completes). No manual invocation needed for standard analysis.
- **D-08:** A `/seraphim:analyze` command also available for on-demand deep analysis outside the pipeline flow.

### Claude's Discretion
- Statistical methods for confidence scoring (simple rates vs z-tests vs bayesian)
- Dashboard technology (Chart.js inline like existing, or evolve if Phase 7 Vercel hosting changes the approach)
- Recommendation presentation format in terminal

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Design Spec
- `docs/specs/2026-04-04-seraphim-v3-design.md` §Adaptive Intelligence — data collection, learning, auto-adjustment, insights dashboard

### Research
- `.planning/research/FEATURES.md` §Adaptive Model Selection — table stakes, differentiators, anti-features (no ML at single-user scale, statistical analysis only)
- `.planning/research/SUMMARY.md` §Expected Features Wave 3 — pattern analysis requires ~20+ runs for meaningful signal; analysis layer can come after logging

### Existing Dashboard
- `~/.claude/dashboard/dashboard.html` — existing Chart.js dashboard pattern (reference for charting approach, NOT to extend)
- `~/.claude/hooks/codex-dashboard-generator.js` — HTML generation pattern with atomic write
- `~/.claude/hooks/codex-global-aggregator.js` — Multi-project JSONL aggregation pattern

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `codex-global-aggregator.js` — Pattern for scanning multiple projects and merging JSONL data
- `codex-dashboard-generator.js` — Self-contained HTML generation with inlined Chart.js
- Chart.js 4.5.1 UMD already cached at `~/.claude/dashboard/assets/`

### Established Patterns
- computeMetrics() pattern: aggregate raw JSONL into dashboard-ready data objects
- buildTimeSeries() with UTC daily buckets and weekly toggle
- generateDashboard returns DASHBOARD_DATA object; HTML rendered from data

### Integration Points
- decisions.jsonl (from Phase 4) is the primary data source
- token-log.jsonl provides cost data
- Dashboard panels read aggregated data, not raw JSONL

</code_context>

<specifics>
## Specific Ideas

No specific requirements — open to standard approaches for statistical analysis and dashboard design.

</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope

</deferred>

---

*Phase: 06-adaptive-intelligence*
*Context gathered: 2026-04-04*
