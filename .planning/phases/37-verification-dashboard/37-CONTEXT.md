# Phase 37: Verification + Dashboard - Context

**Gathered:** 2026-04-10
**Status:** Ready for planning

<domain>
## Phase Boundary

Build verification commands (verify, validate, uat, audit-milestone, audit-uat), dashboard visualization panels (progress bars, velocity chart, roadmap tree, wave progress), and terminal stats command. This is the final phase — it closes the loop from idea capture through verification and visualization.

</domain>

<decisions>
## Implementation Decisions

### Verification Commands
- **D-01:** `/seraphim:verify` traces features to REQ-IDs via REQUIREMENTS.md traceability table + PLAN.md frontmatter. Goal-backward: start from goal, derive must-haves, check codebase. Every report includes at least one REQUIRES_HUMAN_JUDGMENT item.
- **D-02:** `/seraphim:uat` uses persistent UAT.md in phase directory. Accumulates test results across sessions. Each run reads existing state, presents next untested item, records result. YAML frontmatter tracks overall status.
- **D-03:** `/seraphim:validate` spawns nyquist-auditor subagent that reads VERIFICATION.md, identifies coverage gaps, generates additional test scenarios. Writes VALIDATION.md.

### Dashboard Visualization
- **D-04:** Progress bars computed server-side from `.planning/` filesystem — phase completion % from plan summaries, overall milestone from phase count. Rendered as CSS gradient bars in Next.js dashboard.
- **D-05:** Velocity tracking uses rolling 7-day window from git log commit timestamps. Count commits per day, rolling average. Chart.js line chart.
- **D-06:** Roadmap tree view is hierarchical — milestone → phases → waves → tasks → costs. Expandable/collapsible. Data from ROADMAP.md + plan files + session JSONL for costs.
- **D-07:** Wave progress panels show per-wave breakdown within each phase — plans in wave, completion status, agent count.

### Milestone & Cross-Phase Auditing
- **D-08:** `/seraphim:audit-milestone` spawns integration-checker subagent that reads all phase VERIFICATION.md files, checks cross-phase data contracts, validates requirement coverage across the full milestone.
- **D-09:** `/seraphim:audit-uat` scans all UAT.md files across phases, filters for `status: pending` or `status: failed`, presents grouped by phase with links to debug sessions where applicable.

### Terminal Stats
- **D-10:** `/seraphim:stats` shows terminal summary — phases completed/total, plans count, requirements coverage %, git metrics (commits, files changed), timeline (days elapsed, avg days/phase).

### Claude's Discretion
- Dashboard component styling, chart colors, tree expand/collapse UX, and internal subagent prompts at Claude's discretion.

</decisions>

<code_context>
## Existing Code Insights

### Reusable Assets
- `dashboard/` — Next.js app with existing panels (PM roadmap tree, task management, cost charts)
- `dashboard/lib/queries.ts` — Neon query functions
- `dashboard/lib/types.ts` — TypeScript types
- `lib/roadmap.js` — readRoadmap for milestone/feature data
- `lib/requirements.js` — REQ-ID CRUD (Phase 33)
- Existing Chart.js integration in dashboard

### Established Patterns
- `.md` command files with YAML frontmatter
- Subagent dispatch for verification/audit
- Dashboard uses server components with Neon queries
- `.planning/` filesystem as source of truth for progress

### Integration Points
- Commands in `~/.claude/plugins/seraphim/commands/`
- Dashboard panels in `dashboard/app/` and `dashboard/components/`
- `.planning/phases/` for progress computation
- Neon tables for dashboard data

</code_context>

<specifics>
## Specific Ideas

No specific requirements — user gave full Claude discretion.

</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope

</deferred>

---

*Phase: 37-verification-dashboard*
*Context gathered: 2026-04-10 via smart discuss*
