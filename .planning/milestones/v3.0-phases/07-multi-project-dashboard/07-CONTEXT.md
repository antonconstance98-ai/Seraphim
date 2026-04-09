# Phase 7: Multi-Project Dashboard - Context

**Gathered:** 2026-04-04
**Status:** Ready for planning

<domain>
## Phase Boundary

Deliver a Vercel-hosted web application that aggregates progress, metrics, and workflow data across all Seraphim-managed projects. Real-time updates via WebSocket. Full drill-down to rendered phase output files. Seraphim pipeline data only (no v2.0 legacy merge).

</domain>

<decisions>
## Implementation Decisions

### Server Architecture
- **D-01:** Vercel-hosted web app — always live at a URL, not localhost. Deployed to Vercel with proper domain.
- **D-02:** Fallback: if Vercel hosting proves impractical for data sync, fall back to always-on local daemon at 127.0.0.1 with auto-selected port.
- **D-03:** Data sync mechanism: hook fires after each phase completion, pushes metrics and progress data to a Vercel-hosted backend (database via Vercel Marketplace — e.g., Neon Postgres or Upstash Redis).

### Real-Time Updates
- **D-04:** Live push per phase — data pushes after EACH phase completes, not just full pipeline runs. Dashboard reflects progress as phases execute.
- **D-05:** WebSocket connection — dashboard auto-refreshes when new data arrives. See phases completing in real-time without manual browser refresh.

### Data Scope
- **D-06:** Seraphim pipeline data only. Clean slate — no import of v1.0-v2.0 historical data. Old `~/.claude/dashboard/` stays separate and eventually deprecated.
- **D-07:** Data sources per project: `.seraphim/config.json`, `.seraphim/token-log.jsonl`, `.seraphim/decisions.jsonl`, `.seraphim/phases/*/state.json`, phase output files.

### Navigation and Drill-Down
- **D-08:** Full drill-down with rendered phase outputs: overview -> project card -> phase list -> per-phase details (model, tokens, cost, outcome, loops) -> rendered markdown of actual phase output files (vision.md, judgment.md, blueprint.md, crucible.md, etc.)
- **D-09:** Multi-project overview shows: project name, active profile, current phase, progress bar, total cost, last activity date.
- **D-10:** Workflow metrics panel: cross-project model performance, cost trends over time, savings vs Opus-only baseline.

### Claude's Discretion
- Frontend framework choice (Next.js App Router on Vercel is the natural fit)
- Database choice for data persistence (Neon Postgres via Marketplace likely)
- WebSocket implementation (Vercel Functions + WebSocket or external service)
- Dashboard visual design and layout
- Authentication (if any — this is a personal tool, may not need auth)

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Design Spec
- `docs/specs/2026-04-04-seraphim-v3-design.md` — Per-project state structure (`.seraphim/` layout), phase output file naming conventions

### Research
- `.planning/research/ARCHITECTURE.md` — Plugin structure, data flow, component boundaries
- `.planning/research/FEATURES.md` §Multi-Model Orchestration Plugin — decisions.jsonl schema, token logging schema

### Existing Infrastructure
- `~/.claude/hooks/codex-global-aggregator.js` — Multi-project scanning pattern (discovers projects via find, reads JSONL, merges)
- `~/.claude/hooks/codex-dashboard-generator.js` — Dashboard data computation and HTML generation pattern
- `~/.claude/dashboard/` — Existing dashboard structure (reference only, not extending)

### Phase 6 Context
- `.planning/phases/06-adaptive-intelligence/06-CONTEXT.md` — New Seraphim dashboard decisions (separate from v2.0, own identity). Phase 7 dashboard may absorb Phase 6 panels.

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `codex-global-aggregator.js` — Project discovery via `find`, mtime-gated incremental reads, dedup logic
- `codex-dashboard-generator.js` — computeMetrics(), buildTimeSeries(), self-contained HTML generation
- Chart.js 4.5.1 UMD at `~/.claude/dashboard/assets/` — reusable for charting

### Established Patterns
- JSONL append-only logging with session_id correlation
- Atomic file writes via temp+rename
- Multi-project scanning with project-index.json caching

### Integration Points
- Push hook runs after Crucible phase (or any phase completion)
- Reads `.seraphim/` directories across `~/projects/`
- Consumes decisions.jsonl and token-log.jsonl from each project
- Phase output .md files served as rendered markdown in dashboard

</code_context>

<specifics>
## Specific Ideas

- User wants this to be a "multi-project interface and workflow tool" — not just metrics, but a project management view showing what's done, what's left, and current progress across all projects.
- Vercel hosting makes this accessible from any device, not just the workstation terminal.

</specifics>

<deferred>
## Deferred Ideas

- Import v1.0-v2.0 historical data into the new dashboard — user chose clean slate for now
- Authentication/access control — personal tool, may not need it initially
- Mobile-responsive design — focus on desktop first

</deferred>

---

*Phase: 07-multi-project-dashboard*
*Context gathered: 2026-04-04*
