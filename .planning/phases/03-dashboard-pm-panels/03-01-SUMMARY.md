---
phase: 03-dashboard-pm-panels
plan: "01"
subsystem: dashboard
tags: [neon, nextjs, tabbar, pm-tables, queries]
dependency_graph:
  requires: []
  provides: [pm-table-ddl, pm-queries, status-colors, tab-routing]
  affects: [03-02-PLAN, 03-03-PLAN]
tech_stack:
  added: []
  patterns: [url-based-tab-routing, suspense-client-component, neon-sql-template-literals]
key_files:
  created:
    - ~/.claude/plugins/seraphim/dashboard/lib/status-colors.ts
    - ~/.claude/plugins/seraphim/dashboard/components/TabBar.tsx
  modified:
    - ~/.claude/plugins/seraphim/dashboard/scripts/migrate.ts
    - ~/.claude/plugins/seraphim/dashboard/lib/queries.ts
    - ~/.claude/plugins/seraphim/dashboard/app/project/[name]/page.tsx
key_decisions:
  - "TabBar wrapped in Suspense because useSearchParams requires client boundary — Next.js requires Suspense around client components that use useSearchParams in a server page"
  - "searchParams typed as Promise<{tab?: string}> matching Next.js 15 async searchParams API"
metrics:
  duration_minutes: 4
  completed_date: "2026-04-09"
  tasks_completed: 2
  tasks_total: 2
  files_created: 2
  files_modified: 3
requirements: [ROAD-06, TASK-06, OVER-02]
---

# Phase 03 Plan 01: PM Foundation — Tables, Queries, Tab Routing Summary

**One-liner:** Neon PM tables (milestones/features/human_tasks/cost_trend), 5 PM query functions, STATUS_COLORS constant, and URL-based 3-tab navigation on the project detail page with Metrics as default.

## Tasks Completed

| Task | Name | Commit | Files |
|------|------|--------|-------|
| 1 | PM table migration + Neon queries + status colors | 0c29682 | scripts/migrate.ts, lib/queries.ts, lib/status-colors.ts |
| 2 | TabBar component + project page tab routing | b40e73c | components/TabBar.tsx, app/project/[name]/page.tsx |

## Decisions Made

1. **TabBar in Suspense:** Next.js 15 requires Suspense around client components using `useSearchParams` when rendered from a server page. Added `<Suspense fallback={<div className="h-10" />}>` wrapper.

2. **searchParams as Promise:** Next.js 15 async searchParams API requires `Promise<{tab?: string}>` type and `await searchParams` — matched existing `params` pattern already in the page.

3. **PM queries follow getSql() pattern:** All 5 new query functions use `getSql()` from `./db`, consistent with existing getAllProjects/getProjectDecisions/getPhaseStates/getCostTrend functions.

## Verification

- Build: `npx next build` passes with no errors or warnings
- All 4 PM table DDL statements verified in migrate.ts
- All 5 PM query function exports verified in queries.ts
- STATUS_COLORS export verified in status-colors.ts
- TabBar `'use client'`, `useSearchParams`, `role="tablist"` all verified
- Project page `await searchParams`, tab conditionals, Suspense, force-dynamic all verified

## Deviations from Plan

None — plan executed exactly as written.

## Known Stubs

- `{tab === 'plan'}` renders placeholder text "Plan tab — coming in next plan" — intentional, wired in 03-02-PLAN
- `{tab === 'tasks'}` renders placeholder text "Tasks tab — coming in next plan" — intentional, wired in 03-03-PLAN

These stubs do not block the plan's goal (tab routing infrastructure + PM queries ready). Content panels are deferred to subsequent plans by design.

## Self-Check: PASSED

- status-colors.ts: FOUND at ~/.claude/plugins/seraphim/dashboard/lib/status-colors.ts
- TabBar.tsx: FOUND at ~/.claude/plugins/seraphim/dashboard/components/TabBar.tsx
- Commit 0c29682: FOUND
- Commit b40e73c: FOUND
