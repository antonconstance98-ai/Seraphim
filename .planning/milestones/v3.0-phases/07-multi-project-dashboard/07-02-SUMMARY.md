---
phase: 07-multi-project-dashboard
plan: 02
subsystem: database, api
tags: [nextjs, neon, postgres, typescript, tailwind, vercel]

requires:
  - phase: 07-multi-project-dashboard-plan-01
    provides: push tooling that calls /api/ingest

provides:
  - Next.js 16 App Router project scaffolded at dashboard/
  - getSql() lazy Neon client (never crashes next build when DATABASE_URL unset)
  - TypeScript interfaces for Project, Decision, PhaseState, IngestPayload
  - SQL query functions getAllProjects, getProjectDecisions, getPhaseStates, getCostTrend
  - POST /api/ingest endpoint with bearer auth, project upsert, decisions insert, phase_state upsert
  - scripts/migrate.ts idempotent schema creation for all 3 tables

affects: [07-multi-project-dashboard-plan-03, 07-multi-project-dashboard-plan-04]

tech-stack:
  added: [next@16.2.3, @neondatabase/serverless@1.0.2, marked@18, tsx@4, dotenv@17, tailwindcss@4]
  patterns: [getSql lazy singleton pattern, bearer token auth on ingest route, batch decision insert with field validation]

key-files:
  created:
    - dashboard/lib/db.ts
    - dashboard/lib/types.ts
    - dashboard/lib/queries.ts
    - dashboard/app/api/ingest/route.ts
    - dashboard/scripts/migrate.ts
  modified:
    - dashboard/package.json
    - dashboard/app/layout.tsx

key-decisions:
  - "getSql() lazy singleton: neon() never called at module scope — prevents next build crash when DATABASE_URL unset in CI/build environment"
  - "dotenv installed as devDependency (Rule 3 auto-fix): scripts/migrate.ts uses dotenv.config() which requires the package; not bundled by Next.js"
  - "No edge runtime on ingest route: full Node.js serverless function needed for DB writes (edge runtime on plan-04 SSE endpoint only)"
  - "Batch decision insert uses per-record loop not bulk VALUES: neon() tagged template literal syntax requires individual awaits per record for type safety"

patterns-established:
  - "getSql pattern: all DB access in lib/ and app/ imports getSql from lib/db — never calls neon() directly"
  - "Ingest auth: Authorization: Bearer ${DASHBOARD_SECRET} checked before any DB operation, returns 401 on mismatch"

requirements-completed: [DASH-04, DASH-05]

duration: 18min
completed: 2026-04-08
---

# Phase 7 Plan 02: Multi-Project Dashboard Scaffold Summary

**Next.js 16 App Router scaffolded with Neon Postgres schema, lazy getSql() client, typed query functions, and authenticated /api/ingest endpoint for workstation push**

## Performance

- **Duration:** 18 min
- **Started:** 2026-04-08T23:41:00Z
- **Completed:** 2026-04-08T23:59:00Z
- **Tasks:** 2
- **Files modified:** 8

## Accomplishments
- Scaffolded complete Next.js 16 App Router project with TypeScript, Tailwind, and ESLint
- Implemented lazy getSql() pattern ensuring next build never crashes without DATABASE_URL
- Created all four query functions (getAllProjects, getProjectDecisions, getPhaseStates, getCostTrend) with correct Neon tagged-template SQL
- Implemented POST /api/ingest with bearer token auth, project upsert, batch decision insert, and phase_state upsert

## Task Commits

1. **Task 1 + 2: Scaffold, types, DB layer, queries, ingest** - `146c87f` (feat)

**Plan metadata:** (pending docs commit)

## Files Created/Modified
- `dashboard/lib/db.ts` - getSql() lazy Neon client singleton
- `dashboard/lib/types.ts` - Project, Decision, PhaseState, IngestPayload interfaces
- `dashboard/lib/queries.ts` - getAllProjects, getProjectDecisions, getPhaseStates, getCostTrend
- `dashboard/app/api/ingest/route.ts` - POST endpoint with bearer auth + DB upserts
- `dashboard/scripts/migrate.ts` - idempotent CREATE TABLE IF NOT EXISTS for 3 tables
- `dashboard/package.json` - migrate script, dotenv devDep added
- `dashboard/app/layout.tsx` - title updated to "Seraphim Dashboard"

## Decisions Made
- getSql() lazy singleton: neon() never called at module scope — prevents next build crash when DATABASE_URL unset in CI
- dotenv installed as devDependency (Rule 3 auto-fix): migrate script requires dotenv.config()
- No edge runtime on ingest route: full Node.js serverless function needed for DB writes
- Per-record decision insert loop: neon tagged-template syntax requires individual awaits

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 3 - Blocking] Installed missing dotenv devDependency**
- **Found during:** Task 1 (TypeScript verification)
- **Issue:** scripts/migrate.ts imports dotenv but package was not in devDependencies; tsc reported TS2307 cannot find module dotenv
- **Fix:** Ran `npm install -D dotenv`
- **Files modified:** dashboard/package.json, dashboard/package-lock.json
- **Verification:** npx tsc --noEmit passes clean after install
- **Committed in:** 146c87f (combined task commit)

---

**Total deviations:** 1 auto-fixed (1 blocking)
**Impact on plan:** Essential for TypeScript compilation and migrate script operation. No scope creep.

## Issues Encountered
- dashboard/ directory contained a stray seraphim.html file which blocked create-next-app — moved it to the parent seraphim/ directory before scaffolding

## Next Phase Readiness
- All query function signatures are exported from lib/queries.ts — plan 03 UI pages can import without modification
- /api/ingest is deployable; requires DASHBOARD_SECRET and DATABASE_URL environment variables set in Vercel dashboard before first push
- scripts/migrate.ts must be run once against Neon DB before ingest calls will succeed (tables don't exist yet)

---
*Phase: 07-multi-project-dashboard*
*Completed: 2026-04-08*
