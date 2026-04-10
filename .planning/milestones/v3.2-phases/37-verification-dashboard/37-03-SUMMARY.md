---
phase: 37-verification-dashboard
plan: "03"
subsystem: dashboard
tags: [api-routes, components, planning-data, visualization]
dependency_graph:
  requires: []
  provides: [planning-api, velocity-api, roadmap-tree-api, PhaseProgressPanel, WaveProgressPanel]
  affects: [dashboard-progress-tab]
tech_stack:
  added: []
  patterns: [force-dynamic route, Node.js fs reads, execSync git log, server component]
key_files:
  created:
    - ~/.claude/plugins/seraphim/dashboard/app/api/planning/route.ts
    - ~/.claude/plugins/seraphim/dashboard/app/api/velocity/route.ts
    - ~/.claude/plugins/seraphim/dashboard/app/api/roadmap-tree/route.ts
    - ~/.claude/plugins/seraphim/dashboard/components/PhaseProgressPanel.tsx
    - ~/.claude/plugins/seraphim/dashboard/components/WaveProgressPanel.tsx
  modified: []
decisions:
  - "planning/route.ts uses PLANNING_DIR env var with path.join(cwd, '../../.planning') fallback — matches existing plugin pattern"
  - "velocity/route.ts uses Node.js runtime (no edge export) — execSync requires Node, not edge"
  - "roadmap-tree/route.ts returns parse_warnings array — defensive parsing per Pitfall 4"
  - "Both components are server components (no 'use client') — data passed as props from page"
metrics:
  duration: "~5 min"
  completed_date: "2026-04-10"
  tasks_completed: 2
  files_created: 5
  files_modified: 0
---

# Phase 37 Plan 03: Dashboard API Routes and Progress Components Summary

**One-liner:** Three filesystem-backed API routes (planning, velocity, roadmap-tree) and two server components (PhaseProgressPanel, WaveProgressPanel) providing the data layer and progress visualization panels for the dashboard Progress tab.

## Tasks Completed

| Task | Name | Commit | Files |
|------|------|--------|-------|
| 1 | Create three API routes | f45e355 | planning/route.ts, velocity/route.ts, roadmap-tree/route.ts |
| 2 | Create PhaseProgressPanel and WaveProgressPanel | 2e83e18 | PhaseProgressPanel.tsx, WaveProgressPanel.tsx |

## What Was Built

**API Routes (dashboard repo):**

- `/api/planning` — reads `PLANNING_DIR/phases/` directories, counts PLAN.md vs SUMMARY.md files, groups by wave via frontmatter parsing, returns per-phase and milestone-level progress
- `/api/velocity` — executes `git log --since="7 days ago"` via `execSync` in the project git root, returns rolling 7-day commit count per date
- `/api/roadmap-tree` — parses ROADMAP.md for milestone name, reads phase directories and PLAN.md frontmatter, returns hierarchical milestone/phase/wave/plan tree with `parse_warnings` array

**Components (server, no `'use client'`):**

- `PhaseProgressPanel` — milestone summary bar at top (`role="progressbar"`, aria attrs), then one row per phase with `bg-indigo-500 h-1.5 rounded-full` fill bar and `{summaries}/{plans} plans` subscript; empty state with "No phases found" copy
- `WaveProgressPanel` — `bg-white/5 rounded-lg p-4` cards per phase, dot rows per wave using `w-1.5 h-1.5 rounded-full` pattern (`bg-indigo-500` complete, `bg-white/20` incomplete), aria labels on each dot

## Decisions Made

- `planning/route.ts` resolves planning dir via `PLANNING_DIR` env var with `path.join(process.cwd(), '../../.planning')` fallback — consistent with other plugin routes
- `velocity/route.ts` omits `export const runtime = 'edge'` — `execSync` requires Node.js runtime (per Pitfall 5 in RESEARCH.md)
- `roadmap-tree/route.ts` returns `parse_warnings: []` array and continues with partial data on any parse error — defensive parsing per plan spec
- Both components are server components receiving props — data fetching handled by the page that renders them

## Deviations from Plan

None — plan executed exactly as written.

## Known Stubs

None — all components accept real data via props. The page layer (which calls the APIs and passes props) is handled in plan 37-04.

## Self-Check: PASSED

- planning/route.ts: FOUND
- velocity/route.ts: FOUND
- roadmap-tree/route.ts: FOUND
- PhaseProgressPanel.tsx: FOUND
- WaveProgressPanel.tsx: FOUND
- Commits f45e355 and 2e83e18: FOUND in dashboard repo
