---
phase: 07-multi-project-dashboard
plan: "04"
subsystem: dashboard-realtime
tags: [sse, edge-runtime, chart.js, hooks, realtime]
dependency_graph:
  requires: [07-01, 07-03]
  provides: [realtime-sse, metrics-panel, phase-push-hook]
  affects: [dashboard-ui, hook-system]
tech_stack:
  added: [chart.js]
  patterns: [edge-runtime-sse, dynamic-import, fire-and-forget-hook]
key_files:
  created:
    - ~/.claude/plugins/seraphim/dashboard/app/api/events/route.ts
    - ~/.claude/plugins/seraphim/dashboard/hooks/useSSE.ts
    - ~/.claude/plugins/seraphim/dashboard/components/LiveIndicator.tsx
    - ~/.claude/plugins/seraphim/dashboard/components/MetricsPanel.tsx
    - ~/.claude/plugins/seraphim/hooks/phase-push.js
    - ~/.claude/plugins/seraphim/hooks/hooks.json
  modified:
    - ~/.claude/plugins/seraphim/dashboard/app/layout.tsx
    - ~/.claude/plugins/seraphim/dashboard/app/page.tsx
decisions:
  - "Edge Runtime SSE endpoint polls MAX(last_pushed_at) every 5s — correct pattern for Vercel streaming (no WebSocket)"
  - "chart.js loaded via dynamic import inside useEffect — avoids SSR DOM crash per Pitfall 4"
  - "hooks.json created fresh with all three hooks (token-logger, session-start, phase-push) — no prior file existed"
  - "phase-push.js filters to .seraphim/phases/* output files only — prevents push on every Write tool call"
metrics:
  duration: "~12 min"
  completed_date: "2026-04-08"
  tasks_completed: 2
  files_created: 6
  files_modified: 2
---

# Phase 07 Plan 04: Real-Time Layer and Metrics Panel Summary

**One-liner:** Edge Runtime SSE endpoint streaming project activity, Chart.js metrics panel with cost trend and model performance, and phase-push.js hook registered in hooks.json for automatic workstation-to-Vercel data sync.

## Tasks Completed

| Task | Name | Commit | Files |
|------|------|--------|-------|
| 1 | SSE endpoint + useSSE hook + LiveIndicator | 2682418 | route.ts, useSSE.ts, LiveIndicator.tsx, layout.tsx |
| 2 | MetricsPanel + phase-push.js + hooks.json | 35427f7 | MetricsPanel.tsx, phase-push.js, hooks.json, page.tsx |

## What Was Built

### SSE Endpoint (app/api/events/route.ts)
Edge Runtime GET handler that streams `text/event-stream`. Sends an immediate `connected` heartbeat then polls `MAX(last_pushed_at)` from the projects table every 5 seconds. Only emits an `update` event when the value changes. Uses `getSql()` imported inside the function body (not at module scope) to avoid neon() calls at build time.

### useSSE Hook (hooks/useSSE.ts)
Client-side React hook managing EventSource lifecycle. Returns `{ connected, lastEvent }`. Sets `connected=true` on open and on `connected` event type, `connected=false` on error. Closes EventSource on unmount.

### LiveIndicator Component (components/LiveIndicator.tsx)
Client component using useSSE('/api/events'). Renders a green pulsing dot when connected, red dot when disconnected. Shows last push time when an `update` event carries `latest_activity`. Placed in a sticky header in layout.tsx alongside the "Seraphim" title.

### MetricsPanel Component (components/MetricsPanel.tsx)
Client component with two Chart.js canvases loaded via `import('chart.js/auto')` inside useEffect — dynamic import prevents SSR DOM crash. Cost trend: line chart grouping decisions by day. Model performance: dual-axis bar chart showing avg cost (left axis) and success rate % (right axis) per model. Empty state returns a plain-text placeholder. Wired in page.tsx via `allDecisions = projectData.flatMap(({ decisions }) => decisions)`.

### phase-push.js (hooks/phase-push.js)
CommonJS hook script reading stdin for Claude Code PostToolUse event JSON. Extracts `cwd` as project root and `tool_input.file_path` to detect Seraphim phase output files (matches `.seraphim/phases/<phase>/state.json` and the six phase document filenames). Non-matching files exit immediately with `{continue:true}`. Matching files call `pushProjectData(projectRoot, phaseId)` fire-and-forget then exit with `{continue:true}`. Never throws, never blocks.

### hooks.json (hooks/hooks.json)
Created fresh (no prior file). Registers three hooks: token-logger (PostToolUse), session-start (SessionStart), phase-push (PostToolUse with Write matcher).

## Decisions Made

1. **Edge Runtime SSE confirmed correct** — Vercel Serverless Functions close on return; Edge Runtime streaming has no hard timeout. WebSocket rejected per plan constraint.
2. **chart.js dynamic import** — `import('chart.js/auto')` inside useEffect avoids SSR/DOM access during server render (Pitfall 4 from RESEARCH.md).
3. **hooks.json created from scratch** — file did not exist prior to this plan. Included token-logger and session-start entries for completeness alongside new phase-push entry.
4. **phase-push filters to phase output files only** — prevents every Write call from triggering a push. The regex matches the six canonical phase document names and state.json.

## Deviations from Plan

None — plan executed exactly as written. One auto-fix applied: `decisions` field added to the `return` object in projectData Promise.all map (was computed but not returned, causing TypeScript error when flatMap referenced it on line 36).

**[Rule 1 - Bug] Fixed missing decisions field in projectData return**
- Found during: Task 2 (wiring MetricsPanel)
- Issue: `decisions` was computed in the map callback but not included in the returned object, causing TypeScript to error when `projectData.flatMap(({ decisions }) => decisions)` was added
- Fix: Added `decisions` to the return object alongside `project`, `phaseStates`, `totalCost`
- Files modified: dashboard/app/page.tsx
- Commit: 35427f7

## Self-Check: PASSED

All 6 created files confirmed on disk. Both task commits (2682418, 35427f7) confirmed in git log.
