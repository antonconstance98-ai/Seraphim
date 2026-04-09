---
phase: 03-dashboard-pm-panels
plan: "03"
subsystem: dashboard-ui
tags: [pm-panels, project-card, milestone, sparkline, chart-js]
dependency_graph:
  requires: [03-01]
  provides: [OVER-02]
  affects: [dashboard-home]
tech_stack:
  added: []
  patterns: [server-component, client-component, chart-js-dynamic-import]
key_files:
  created:
    - ~/.claude/plugins/seraphim/dashboard/components/MilestoneProgressBar.tsx
    - ~/.claude/plugins/seraphim/dashboard/components/NeedsAttentionBadge.tsx
    - ~/.claude/plugins/seraphim/dashboard/components/CostSparkline.tsx
  modified:
    - ~/.claude/plugins/seraphim/dashboard/components/ProjectCard.tsx
    - ~/.claude/plugins/seraphim/dashboard/app/page.tsx
decisions:
  - "CostSparkline uses dynamic import('chart.js/auto') inside useEffect to avoid SSR issues"
  - "NeedsAttentionBadge returns null when no attention needed — no empty DOM nodes"
  - "MilestoneProgressBar priority: in-progress > planned > last milestone"
metrics:
  duration: "8 minutes"
  completed_date: "2026-04-09"
  tasks_completed: 2
  tasks_total: 2
  files_created: 3
  files_modified: 2
---

# Phase 03 Plan 03: PM Card Components Summary

**One-liner:** Cross-project PM overview on home page via MilestoneProgressBar, NeedsAttentionBadge, and Chart.js CostSparkline wired into ProjectCard.

## What Was Built

Three new UI components extend each ProjectCard on the home page with PM status at a glance:

- **MilestoneProgressBar** (server component): Shows the active milestone's name and feature completion as an indigo progress bar. Selects active milestone by priority: in-progress > planned > last.
- **NeedsAttentionBadge** (server component): Renders a red "Needs attention" badge when any task has urgency=high and status!=done, or any milestone is blocked.
- **CostSparkline** (client component): Renders a 96x48 Chart.js mini line chart of cost trend over time. Uses dynamic import to avoid SSR, destroys chart on cleanup.

**ProjectCard** extended with three optional props (`milestones`, `pmTasks`, `costTrend`) defaulting to empty arrays for full backward compatibility.

**page.tsx** now fetches `getProjectPmSummary` for each project in the existing `Promise.all` and spreads the result into projectData, then passes milestone/task/trend props to each ProjectCard.

## Requirement Fulfilled

OVER-02: Dashboard cross-project panel shows PM status per project — milestone progress, feature counts, human tasks pending, cost trend sparkline, and needs-attention badge on projects with blockers.

## Deviations from Plan

None — plan executed exactly as written.

## Self-Check

- [x] MilestoneProgressBar.tsx created and exports `MilestoneProgressBar`
- [x] NeedsAttentionBadge.tsx created and exports `NeedsAttentionBadge`
- [x] CostSparkline.tsx created with `'use client'`, chart cleanup, aria-hidden
- [x] ProjectCard.tsx imports all three components
- [x] page.tsx calls `getProjectPmSummary` and passes props
- [x] `npx next build` passes with no errors
- [x] Commits: 4c63537 (Task 1), f7d5fae (Task 2)

## Self-Check: PASSED
