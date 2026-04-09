# Phase 3: Dashboard PM Panels - Context

**Gathered:** 2026-04-09
**Status:** Ready for planning

<domain>
## Phase Boundary

Add PM panels to the web dashboard: roadmap visualization, human task management, and cross-project overview. After this phase, the dashboard becomes the human's PM command center -- all project management visible at a glance in the browser.

</domain>

<decisions>
## Implementation Decisions

### Dashboard Navigation (D-01)
- **D-01:** Tabbed project page with 3 tabs: **Metrics | Plan | Tasks**. Metrics = existing v3.0 pipeline view (costs, tokens, sessions). Plan = roadmap/milestone/feature tree. Tasks = human tasks grouped by type. Cross-project overview goes on the home page.

### Roadmap Panel (D-02)
- **D-02:** Plan tab shows milestone-feature tree with visual status indicators. Status colors: planned (gray), in-progress (blue), complete (green), blocked (red/amber). Feature entries show pipeline position (which of 6 phases complete).

### Human Tasks Panel (D-03)
- **D-03:** Tasks tab shows all human tasks for the project grouped by type (decision, research, review, validation, skills). Each task shows status, urgency, and linked feature. Matches the terminal inbox layout (by project then type).

### Cross-Project Panel (D-04)
- **D-04:** Home page extended with PM summary per project card: milestone progress bar, feature count (complete/total), pending human task count, cost trend sparkline. "Needs attention" badge on projects with blockers.

### Tab Implementation (D-05)
- **D-05:** Tabs are URL-based (e.g., `/project/[name]?tab=plan`) for shareability and deep linking. Default tab is Metrics (preserves v3.0 behavior). Server Components render tab content from Neon queries.

### Claude's Discretion
- Exact Tailwind styling and component layout within each tab
- Chart.js configuration for cost trend sparklines
- Responsive breakpoints and mobile layout
- Loading states and error boundaries per tab

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Existing Dashboard Code
- `~/.claude/plugins/seraphim/dashboard/app/page.tsx` -- Home page with ProjectCard; extend with PM summary
- `~/.claude/plugins/seraphim/dashboard/app/project/[name]/page.tsx` -- Project detail; add tab navigation
- `~/.claude/plugins/seraphim/dashboard/components/ProjectCard.tsx` -- Extend with milestone progress, task count
- `~/.claude/plugins/seraphim/dashboard/components/TaskList.tsx` -- Existing task list; reuse for Tasks tab
- `~/.claude/plugins/seraphim/dashboard/lib/db.ts` -- Neon queries; add PM table queries

### Phase 1-2 Context
- `.planning/phases/01-core-pm-primitives/01-CONTEXT.md` -- roadmap.json schema, feature IDs
- `.planning/phases/02-progress-visibility/02-CONTEXT.md` -- Neon tables, sync strategy

### Requirements
- `.planning/REQUIREMENTS.md` -- ROAD-06, TASK-06, OVER-02

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `ProjectCard.tsx`: Server Component with human task badge -- extend with milestone progress
- `TaskList.tsx`: Human (amber cards) / AI (table) task views -- reuse for Tasks tab
- `page.tsx` (project detail): Phase roadmap, session history -- becomes Metrics tab content
- Chart.js: Already loaded via dynamic import in useEffect -- reuse for sparklines

### Established Patterns
- Server Components for data pages (no `use client` unless hooks needed)
- `force-dynamic` on DB-querying pages (avoids prerender crash without DATABASE_URL)
- Chart.js via CDN/dynamic import inside useEffect (avoids SSR DOM crash)
- Neon queries in page server functions (getSql() lazy singleton)

### Integration Points
- Project detail page gets tab wrapper component
- Home page ProjectCard gets PM summary fields
- New Neon queries for milestones, features, human_tasks tables (from Phase 2)
- URL query param `?tab=plan|tasks|metrics` for tab routing

</code_context>

<specifics>
## Specific Ideas

- Tab names: Metrics | Plan | Tasks (user chose shorter labels)
- Cross-project overview on home page, not a separate page
- Cost trend sparklines in project cards on home page

</specifics>

<deferred>
## Deferred Ideas

None -- discussion stayed within phase scope

</deferred>

---

*Phase: 03-dashboard-pm-panels*
*Context gathered: 2026-04-09*
