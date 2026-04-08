---
phase: 07-multi-project-dashboard
plan: "03"
subsystem: dashboard-ui
tags: [nextjs, react, tailwind, server-components, dashboard]
dependency_graph:
  requires: [07-02]
  provides: [DASH-05, DASH-06]
  affects: [dashboard-pages]
tech_stack:
  added: []
  patterns: [server-components, async-params, dynamic-import, parallel-fetch]
key_files:
  created:
    - ~/.claude/plugins/seraphim/dashboard/components/ProjectCard.tsx
    - ~/.claude/plugins/seraphim/dashboard/components/PhaseRoadmap.tsx
    - ~/.claude/plugins/seraphim/dashboard/components/MarkdownRenderer.tsx
    - ~/.claude/plugins/seraphim/dashboard/app/project/[name]/page.tsx
  modified:
    - ~/.claude/plugins/seraphim/dashboard/app/page.tsx
    - ~/.claude/plugins/seraphim/dashboard/app/globals.css
decisions:
  - "ProjectCard is a Server Component (no hooks needed) — no 'use client' directive added"
  - "parallel Promise.all in page.tsx fetches phaseStates + decisions simultaneously for each project"
  - "groupIntoRuns detects new runs by phase === 'discover' transition — matches existing decisions-logger schema"
metrics:
  duration: "12 minutes"
  completed_date: "2026-04-08"
  tasks_completed: 2
  files_changed: 6
---

# Phase 07 Plan 03: Dashboard Pages and Components Summary

**One-liner:** Dark-themed Next.js App Router dashboard with project overview grid, per-project phase roadmap, run history table, and dynamic markdown renderer using marked.

## What Was Built

### Task 1: ProjectCard + Multi-project Overview

**ProjectCard component** (`components/ProjectCard.tsx`) renders:
- Project name as a link to `/project/[name]`
- Profile badge with color-coded pill (indigo=performance, teal=balanced, amber=budget)
- Progress bar counting completed PhaseStates out of 6 with "N/6 phases" label
- Current phase derived from first incomplete phase in canonical order
- Total cost formatted to 4 decimal places
- Relative last activity time via inline `relativeTime()` helper

**app/page.tsx** is a pure server component that:
- Calls `getAllProjects()` then fans out `Promise.all` per project for phaseStates + decisions
- Sums `cost_usd` from decisions for each project's `totalCost`
- Shows empty state with instructional message when no projects found

**app/globals.css** updated: body background `#0d0d0d`, color `#e5e7eb`, `ui-monospace` font family.

### Task 2: PhaseRoadmap + MarkdownRenderer + Drill-down Page

**PhaseRoadmap component** (`components/PhaseRoadmap.tsx`) renders all six phases (discover → envision → judge → architect → forge → crucible) with:
- ✓ green-400 for completed, ⟳ indigo-400 for in-progress, ○ gray-500 for pending
- Completion date when available
- Aggregate loop and retry counts from PhaseState records

**MarkdownRenderer component** (`components/MarkdownRenderer.tsx`):
- `'use client'` directive with `useEffect` + `useState`
- Dynamic `import('marked')` to avoid SSR bundle bloat
- `dangerouslySetInnerHTML` acceptable — content is user's own local .md files

**app/project/[name]/page.tsx** server component:
- `await params` pattern (Next.js 15+ async params requirement)
- `decodeURIComponent` on the name segment
- `Promise.all` for parallel phaseStates + decisions fetch
- Sections: Phase Roadmap, Pipeline Run History table, Human Tasks, AI Activity (last 20)
- Run grouping: new run starts on `phase === 'discover'` after prior records
- Not-found state when both arrays are empty

## Deviations from Plan

None — plan executed exactly as written.

## Known Stubs

None — all data sources are wired to real DB query functions from plan 02.

## Self-Check: PASSED

Files created:
- /home/alucardmessangeroflight/.claude/plugins/seraphim/dashboard/components/ProjectCard.tsx — FOUND
- /home/alucardmessangeroflight/.claude/plugins/seraphim/dashboard/components/PhaseRoadmap.tsx — FOUND
- /home/alucardmessangeroflight/.claude/plugins/seraphim/dashboard/components/MarkdownRenderer.tsx — FOUND
- /home/alucardmessangeroflight/.claude/plugins/seraphim/dashboard/app/project/[name]/page.tsx — FOUND

Commits:
- 9497072 — feat(07-03): ProjectCard component and multi-project overview page
- a0e9638 — feat(07-03): PhaseRoadmap, MarkdownRenderer, and per-project drill-down page
