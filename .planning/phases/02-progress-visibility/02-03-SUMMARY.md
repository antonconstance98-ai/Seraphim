---
phase: 02-progress-visibility
plan: "03"
subsystem: dashboard-ingest
tags: [neon, ingest, pm-data, types, backward-compat]
dependency_graph:
  requires: []
  provides: [pm-ingest-endpoint, pm-types]
  affects: [dashboard-api, push-client]
tech_stack:
  added: []
  patterns: [upsert-on-conflict, optional-payload-fields]
key_files:
  created: []
  modified:
    - ~/.claude/plugins/seraphim/dashboard/lib/types.ts
    - ~/.claude/plugins/seraphim/dashboard/app/api/ingest/route.ts
decisions:
  - "PM fields are fully optional in IngestPayload -- existing callers (push-client.js) require no changes"
  - "cost_trend table added to DDL scope alongside the 3 primary PM tables"
metrics:
  duration: "5min"
  completed_date: "2026-04-09T18:02:10Z"
  tasks_completed: 1
  tasks_total: 2
  files_modified: 2
---

# Phase 02 Plan 03: Neon PM Schema and Ingest Extension Summary

Extended the dashboard ingest endpoint and TypeScript types to accept optional PM payloads (milestones, features, human_tasks, cost_trend) with upsert-on-conflict semantics, fully backward compatible with existing push-client callers.

## Tasks Completed

| Task | Name | Commit | Files |
|------|------|--------|-------|
| 2 | Extend IngestPayload type and ingest route for PM data | 55d5522 | dashboard/lib/types.ts, dashboard/app/api/ingest/route.ts |

## Tasks Deferred

| Task | Name | Reason |
|------|------|--------|
| 1 | Apply Neon DDL for PM tables | Checkpoint: requires manual DDL application in Neon Console (auto-approved in autonomous mode) |

## Deviations from Plan

**Auto-approved checkpoint (Task 1):** Running in autonomous mode — DDL checkpoint auto-approved. The SQL was not applied by this agent. See DDL section below for the exact SQL to run.

## Known Stubs

None — the ingest route PM blocks are fully wired. They will silently no-op until the DDL is applied and a push-client sends PM fields.

## Pending Manual Action Required

The following DDL must be applied to the Neon database before PM ingest will work. Run via Neon Console SQL Editor or `psql $DATABASE_URL`:

```sql
CREATE TABLE IF NOT EXISTS milestones (
  id             SERIAL PRIMARY KEY,
  project_name   TEXT NOT NULL,
  version        TEXT NOT NULL,
  name           TEXT NOT NULL,
  status         TEXT NOT NULL,
  feature_count  INTEGER NOT NULL DEFAULT 0,
  complete_count INTEGER NOT NULL DEFAULT 0,
  cost_usd       NUMERIC(10, 4) NOT NULL DEFAULT 0,
  synced_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  UNIQUE (project_name, version)
);

CREATE TABLE IF NOT EXISTS features (
  id               SERIAL PRIMARY KEY,
  project_name     TEXT NOT NULL,
  feature_id       TEXT NOT NULL,
  slug             TEXT,
  name             TEXT NOT NULL,
  status           TEXT NOT NULL,
  milestone_version TEXT NOT NULL,
  pipeline_phase   TEXT,
  cost_usd         NUMERIC(10, 4) NOT NULL DEFAULT 0,
  synced_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  UNIQUE (project_name, feature_id)
);

CREATE TABLE IF NOT EXISTS human_tasks (
  id           SERIAL PRIMARY KEY,
  project_name TEXT NOT NULL,
  task_id      TEXT NOT NULL,
  type         TEXT NOT NULL,
  status       TEXT NOT NULL,
  feature_id   TEXT,
  urgency      TEXT NOT NULL DEFAULT 'normal',
  synced_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  UNIQUE (project_name, task_id)
);

CREATE TABLE IF NOT EXISTS cost_trend (
  id           SERIAL PRIMARY KEY,
  project_name TEXT NOT NULL,
  date         TEXT NOT NULL,
  cost_usd     NUMERIC(10,4) NOT NULL DEFAULT 0,
  synced_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  UNIQUE (project_name, date)
);
```

## Self-Check: PASSED

- FOUND: dashboard/lib/types.ts
- FOUND: dashboard/app/api/ingest/route.ts
- FOUND: commit 55d5522
