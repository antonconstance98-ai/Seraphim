---
phase: 32-foundations
plan: "01"
subsystem: dashboard/migrations
tags: [neon, ddl, schema, migrations, project_name]
dependency_graph:
  requires: []
  provides: [FOUND-01, FOUND-02]
  affects: [dashboard/lib/queries.ts, dashboard/app/api/ingest/route.ts]
tech_stack:
  added: []
  patterns: [idempotent-ddl, CREATE TABLE IF NOT EXISTS, conditional RENAME via DO block]
key_files:
  created:
    - dashboard/migrations/001-initial-pm-schema.sql
    - dashboard/migrations/migrate.ts
  modified:
    - dashboard/lib/queries.ts
    - dashboard/app/api/ingest/route.ts
decisions:
  - "migrations/ dir lives in plugin repo at ~/.claude/plugins/seraphim/dashboard/migrations/ — project root has no dashboard/ dir"
  - "splitStatements() handles DO $$ dollar-quote blocks to avoid splitting mid-block on semicolons"
  - "migrate.ts uses sql.unsafe() to execute raw statement strings — necessary since getSql() returns tagged template function"
metrics:
  duration: ~12 min
  completed_date: "2026-04-09"
  tasks_completed: 2
  tasks_total: 2
  files_created: 2
  files_modified: 2
---

# Phase 32 Plan 01: Neon DDL Migration Infrastructure Summary

Idempotent DDL for all 7 PM tables committed to dashboard/migrations/ with project_name column standardization across queries and ingest route.

## Tasks Completed

| Task | Name | Commit | Files |
|------|------|--------|-------|
| 1 | Create migration SQL and runner script | cb3fc58 | dashboard/migrations/001-initial-pm-schema.sql, dashboard/migrations/migrate.ts |
| 2 | Update queries.ts and ingest route for project_name consistency | eb900f9 | dashboard/lib/queries.ts, dashboard/app/api/ingest/route.ts |

## What Was Built

**Task 1 — Migration infrastructure:**
- `001-initial-pm-schema.sql`: Idempotent DDL for 7 tables: `projects`, `decisions`, `phase_states`, `milestones`, `features`, `human_tasks`, `cost_trend`. All PM tables use `project_name` column. `decisions` table includes `feature_id TEXT` column (prepares for FOUND-03). Ends with a `DO $$ BEGIN ... END $$` block that conditionally renames any existing `project` column to `project_name` in the four PM tables.
- `migrate.ts`: Reads SQL file, splits on statement boundaries (handles `DO $$ ... END $$` dollar-quoted blocks), executes each statement via `getSql()`. Gates on `DATABASE_URL` at startup with clear instructions pointing to Neon Console.

**Task 2 — Schema unification:**
- `queries.ts`: Updated 4 functions (`getMilestones`, `getFeatures`, `getHumanTasks`, `getProjectCostTrend`) from `WHERE project = ${projectName}` to `WHERE project_name = ${projectName}`. Zero old refs remain.
- `ingest/route.ts`: Updated 4 INSERT statements (milestones, features, human_tasks, cost_trend) — column lists changed from `project` to `project_name`, ON CONFLICT clauses updated to match.

## Verification Results

- `grep -c "WHERE project = " dashboard/lib/queries.ts` → 0 (no old refs)
- `grep -c "CREATE TABLE IF NOT EXISTS" 001-initial-pm-schema.sql` → 8 (7 tables + 1 comment line)
- `grep -c "project_name" 001-initial-pm-schema.sql` → 16 (all PM tables use project_name)
- All ON CONFLICT clauses in ingest route updated to project_name

## Deviations from Plan

### Context Discovery

**1. [Rule 3 - Blocking] Dashboard in plugin repo, not project root**
- **Found during:** Task 1 setup
- **Issue:** Plan refers to `dashboard/migrations/` but the seraphim project root has no `dashboard/` directory. The dashboard lives in the plugin repo at `~/.claude/plugins/seraphim/dashboard/`.
- **Fix:** Created files in the correct location (`~/.claude/plugins/seraphim/dashboard/migrations/`) and committed to the plugin repo's git history.
- **Impact:** No behavior change — files are in the right place for the migration runner to work.

### Pre-existing Work

**2. Context — feature_id partially applied by prior agent**
- A previous agent (commit `9c07ba7`) had already added `feature_id` to the decisions INSERT in route.ts. The ingest route read during Task 2 reflected this. No regression — Task 2 only touched the PM table INSERTs (milestones, features, human_tasks, cost_trend).

## Known Stubs

None. All changes are functional DDL and query updates. No UI rendering paths affected.

## Requirements Satisfied

- FOUND-01: Migration SQL committed — all 7 PM tables can now be created in Neon by running `npx tsx dashboard/migrations/migrate.ts`
- FOUND-02: `project` vs `project_name` mismatch resolved — queries return rows after ingest

## Self-Check: PASSED

All created files confirmed present. All commits (cb3fc58, eb900f9) confirmed in plugin repo git log.
