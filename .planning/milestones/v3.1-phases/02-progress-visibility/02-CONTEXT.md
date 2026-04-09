# Phase 2: Progress Visibility - Context

**Gathered:** 2026-04-09
**Status:** Ready for planning

<domain>
## Phase Boundary

Cross-project oversight from terminal, data infrastructure for dashboard, feature dependencies, blocked feature signals, and cost trend aggregation. After this phase, the developer sees all projects at a glance and the Neon database has all PM data ready for dashboard rendering.

</domain>

<decisions>
## Implementation Decisions

### Attention Signals (D-01)
- **D-01:** "What needs attention" is integrated into `/seraphim:overview` output as a highlighted section at the top, NOT a separate command. Surfaces: blocked features, exceeded WIP limits, pending human gates.

### Neon Sync Strategy (D-02)
- **D-02:** Both auto-push via hooks AND manual `/seraphim:sync` command. Auto-push extends `phase-push.js` to detect roadmap.json and human task changes. Manual sync provides force-refresh. Same fire-and-forget pattern as v3.0.

### Neon Tables (D-03)
- **D-03:** Three new tables (additive, no existing table changes): `milestones` (project, version, name, status, feature_count, complete_count, cost), `features` (project, feature_id, slug, name, status, milestone_version, pipeline_phase, cost), `human_tasks` (project, task_id, type, status, feature_id, urgency).

### Feature Dependencies (D-04)
- **D-04:** `depends_on` array in feature schema holds feature IDs. `/seraphim:start` checks dependencies and warns (not blocks) if incomplete. Warning includes which dependencies are missing.

### Skills and Research Tasks (D-05)
- **D-05:** Skills development tasks have `type: skills` with `domain` field and recommended resources. Research tasks have `type: research` with context injection -- on completion, notes auto-index to project knowledge via existing RAG tools.

### Cost Trend Aggregation (D-06)
- **D-06:** Cross-project cost trend aggregates decisions.jsonl across all projects by date. Grouped daily. Data pushed to Neon via sync script for dashboard consumption.

### Claude's Discretion
- Exact SQL schema and column types for Neon tables
- Sync script implementation details (batch vs individual inserts)
- Overview terminal output formatting and column layout

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Existing Plugin Code
- `~/.claude/plugins/seraphim/lib/multi-project-scanner.js` -- Extend to include PM data in scan results
- `~/.claude/plugins/seraphim/lib/push-client.js` -- Fire-and-forget Neon sync pattern; extend for PM tables
- `~/.claude/plugins/seraphim/hooks/phase-push.js` -- Hook that triggers push; extend to detect PM file changes
- `~/.claude/plugins/seraphim/dashboard/lib/db.ts` -- Lazy getSql() Neon singleton
- `~/.claude/plugins/seraphim/dashboard/app/api/ingest/route.ts` -- Existing POST endpoint; extend for PM data

### Phase 1 Context
- `.planning/phases/01-core-pm-primitives/01-CONTEXT.md` -- roadmap.json schema, feature IDs, inbox layout

### Requirements
- `.planning/REQUIREMENTS.md` -- QUEUE-05, TASK-05, TASK-07, OVER-01, OVER-03, OVER-04, OVER-05, ARCH-04, ARCH-05

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `multi-project-scanner.js`: Already scans ~/projects/ for Seraphim state -- extend output schema
- `push-client.js`: Fire-and-forget Neon push -- reuse pattern for PM tables
- `phase-push.js`: PostToolUse hook filtering .seraphim/ file changes -- extend glob pattern
- `dashboard/lib/db.ts`: getSql() lazy singleton -- reuse for new table queries

### Established Patterns
- Neon ingest: POST to /api/ingest with Bearer auth, upserts via SQL
- Sync: local files -> JSONL scan -> HTTP POST -> Neon upsert
- Dashboard: Server Components read from Neon, no client-side state

### Integration Points
- `/api/ingest/route.ts` -- Add PM data handling (milestones, features, human_tasks payloads)
- `phase-push.js` -- Extend file pattern matcher to include roadmap.json changes
- `multi-project-scanner.js` -- Add PM fields to scan output

</code_context>

<specifics>
## Specific Ideas

- Attention signals at top of overview output -- most important info first
- Both auto and manual sync -- belt and suspenders approach
- Skills task type carries domain context for human development tracking

</specifics>

<deferred>
## Deferred Ideas

None -- discussion stayed within phase scope

</deferred>

---

*Phase: 02-progress-visibility*
*Context gathered: 2026-04-09*
