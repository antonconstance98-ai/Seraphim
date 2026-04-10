---
phase: 36-human-tasks-debugging
plan: "01"
subsystem: human-tasks
tags: [human-tasks, schema, push-client, ingest, inbox]
dependency_graph:
  requires: []
  provides: [human-task-enrichment-schema, push-client-enrichment, ingest-enrichment, inbox-enrichment-display]
  affects: [dashboard/migrations, lib/push-client.js, dashboard/app/api/ingest/route.ts, commands/inbox.md]
tech_stack:
  added: []
  patterns: [nullable-column-migration, optional-payload-fields, conditional-inline-display]
key_files:
  created:
    - dashboard/migrations/002-human-tasks-enrichment.sql
  modified:
    - lib/push-client.js
    - dashboard/app/api/ingest/route.ts
    - commands/inbox.md
decisions:
  - "skills_to_learn stored as comma-separated string in marker attrs; split on display"
  - "Used ?? null (not || null) for enrichment field fallbacks in ingest route to distinguish empty string from absent"
metrics:
  duration_min: 5
  completed_date: "2026-04-10"
  tasks_completed: 2
  files_modified: 4
---

# Phase 36 Plan 01: Human Task Enrichment Pipeline Summary

**One-liner:** End-to-end enrichment pipeline adding skills_to_learn, thought_prompt, and research_task fields from Neon schema through push-client extraction to inbox coaching display.

## What Was Built

Three optional coaching fields added across the full human task data path:

1. **Migration SQL** (`dashboard/migrations/002-human-tasks-enrichment.sql`) — Three `IF NOT EXISTS` ALTER TABLE statements adding nullable TEXT columns to `human_tasks`.

2. **push-client enrichment** (`lib/push-client.js`) — `scanPendingTasks` now reads `skills_to_learn`, `thought_prompt`, `research_task` from HUMAN_TASK marker attributes using the existing `get(key)` regex pattern and includes them in the returned task objects.

3. **Ingest route** (`dashboard/app/api/ingest/route.ts`) — INSERT/UPSERT for `human_tasks` extended with all three columns, using `?? null` fallback, and ON CONFLICT DO UPDATE includes all three new columns.

4. **Inbox display** (`commands/inbox.md`) — Task display loop conditionally renders Skills as bracketed tags (comma-split), Think prompt, and Research task. Absent fields produce no output.

## Commits

- `3a02d05`: feat(36-01): add human task enrichment fields to schema, push-client, and ingest route
- `ff98305`: feat(36-01): display enrichment fields inline in inbox command

## Deviations from Plan

None — plan executed exactly as written.

## Known Stubs

None.
