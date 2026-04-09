---
phase: 07-multi-project-dashboard
plan: 05
subsystem: infra
tags: [vercel, neon, postgres, nextjs, deployment, ci]

requires:
  - phase: 07-multi-project-dashboard/07-04
    provides: "Next.js dashboard app with SSE endpoint, MetricsPanel, phase-push hook — fully built and locally tested"

provides:
  - ".vercelignore excluding node_modules, .env.local, scripts/, and logs from deployment"
  - "force-dynamic directive on root page and project detail page — prevents build-time DB crash"
  - "npm run build exits 0 cleanly, ready for vercel deploy"
  - "CHECKPOINT: Vercel login, Neon provision, DB migration, production deploy, env var setup — awaiting human execution"

affects:
  - push-client.js (needs SERAPHIM_DASHBOARD_URL env var)
  - phase-push.js hook (needs SERAPHIM_DASHBOARD_SECRET env var)
  - 08-thought-orphanage-integration (will push to same Neon DB)

tech-stack:
  added: [vercel-cli]
  patterns:
    - "export const dynamic = 'force-dynamic' on all pages that query DB — prevents static prerender crash at build time when DATABASE_URL unset"

key-files:
  created:
    - dashboard/.vercelignore
  modified:
    - dashboard/app/page.tsx
    - dashboard/app/project/[name]/page.tsx

key-decisions:
  - "force-dynamic on DB pages — neon() is lazy via getSql() but Next.js static prerender still fires the page function without DATABASE_URL; force-dynamic skips prerender entirely"
  - "vercel integration add neon — preferred over manual Neon account creation; auto-injects DATABASE_URL into Vercel project env, no copy-paste error risk"

patterns-established:
  - "All Next.js pages that call getSql() must declare export const dynamic = 'force-dynamic'"

requirements-completed: [DASH-01, DASH-02, DASH-03, DASH-04, DASH-05, DASH-06, DASH-07]

duration: 8min
completed: 2026-04-08
---

# Phase 7 Plan 05: Vercel Deployment and Neon Provisioning Summary

**Next.js dashboard build cleaned and pre-deploy artifacts created; deployment checkpoint reached pending Vercel login, Neon integration, DB migration, and production URL setup**

## Performance

- **Duration:** ~8 min
- **Started:** 2026-04-08T00:00:00Z
- **Completed:** 2026-04-08 (partial — checkpoint reached)
- **Tasks:** 1 of 2 (Task 2 is a human-action checkpoint)
- **Files modified:** 3

## Accomplishments

- Created `.vercelignore` to exclude node_modules, .env.local, scripts/, and logs from deployment
- Fixed build-time DB crash by adding `export const dynamic = 'force-dynamic'` to both DB-querying pages
- `npm run build` now exits 0 cleanly — all routes compile, TypeScript passes
- Vercel CLI installed globally (`vercel` command available)
- Reached deployment checkpoint — all automated pre-deploy work is done

## Task Commits

1. **Task 1: Pre-deploy preparation and .vercelignore** - `17d14d2` (feat)

## Files Created/Modified

- `dashboard/.vercelignore` - Excludes node_modules, .env.local, scripts/, *.log from deployment
- `dashboard/app/page.tsx` - Added `export const dynamic = 'force-dynamic'`
- `dashboard/app/project/[name]/page.tsx` - Added `export const dynamic = 'force-dynamic'`

## Decisions Made

- `export const dynamic = 'force-dynamic'` on all pages that call `getSql()` — Next.js static prerender runs the page function at build time without DATABASE_URL set; the lazy getSql() singleton still calls neon() with undefined, which throws. force-dynamic opts the route out of static generation entirely.

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 1 - Bug] Build-time DB crash on static prerender**
- **Found during:** Task 1 (npm run build)
- **Issue:** `app/page.tsx` and `app/project/[name]/page.tsx` call `getSql()` → `neon()` during Next.js static prerender at build time when DATABASE_URL is not set in the build environment
- **Fix:** Added `export const dynamic = 'force-dynamic'` to both pages, forcing server-render on demand instead of static prerender
- **Files modified:** dashboard/app/page.tsx, dashboard/app/project/[name]/page.tsx
- **Verification:** `npm run build` exits 0; both pages show `ƒ (Dynamic)` in route table
- **Committed in:** 17d14d2 (Task 1 commit)

---

**Total deviations:** 1 auto-fixed (Rule 1 - bug)
**Impact on plan:** Essential for deployment. No scope creep.

## Issues Encountered

None beyond the build crash, which was anticipated in the plan and auto-fixed.

## User Setup Required

**Deployment requires manual steps.** Complete these in order:

**Step 1 — Authenticate Vercel CLI:**
```bash
vercel whoami || vercel login
```

**Step 2 — Deploy preview:**
```bash
cd ~/.claude/plugins/seraphim/dashboard
vercel deploy --yes 2>&1
```

**Step 3 — Provision Neon Postgres:**
```bash
cd ~/.claude/plugins/seraphim/dashboard
vercel integration add neon --yes 2>&1
```

**Step 4 — Set DASHBOARD_SECRET:**
```bash
SECRET=$(node -e "console.log(require('crypto').randomBytes(32).toString('hex'))")
echo "Your secret: $SECRET"
vercel env add DASHBOARD_SECRET production <<< "$SECRET"
```

**Step 5 — Pull env vars and run DB migration:**
```bash
cd ~/.claude/plugins/seraphim/dashboard
vercel env pull .env.local --yes
npm run migrate
```

**Step 6 — Deploy to production:**
```bash
vercel deploy --prod --yes 2>&1
```

**Step 7 — Test ingest endpoint:**
```bash
PROD_URL="<paste production URL>"
curl -s -X POST "$PROD_URL/api/ingest" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $SECRET" \
  -d '{"project_name":"test","root_path":"/tmp/test","profile":"performance","phase_id":"discover","phase_state":{"phaseId":"discover","loops":{},"retries":{},"completed":true,"completed_at":"2026-04-08T00:00:00.000Z"},"decisions":[],"pushed_at":"2026-04-08T00:00:00.000Z"}' \
  | python3 -m json.tool
```
Expected: `{"ok": true, "project": "test", "phase": "discover"}`

**Step 8 — Set env vars on workstation:**
```bash
echo "export SERAPHIM_DASHBOARD_URL='$PROD_URL'" >> ~/.bashrc
echo "export SERAPHIM_DASHBOARD_SECRET='$SECRET'" >> ~/.bashrc
source ~/.bashrc
```

**Checklist before marking plan complete:**
- [ ] `vercel deploy --prod` succeeded with no build errors
- [ ] DB migration printed "Migration complete"
- [ ] /api/ingest returned `{"ok": true}` with Bearer token
- [ ] /api/ingest returned 401 without Bearer token
- [ ] Dashboard URL loads in browser
- [ ] SERAPHIM_DASHBOARD_URL and SERAPHIM_DASHBOARD_SECRET in ~/.bashrc

## Next Phase Readiness

- Pre-deploy work is done. Dashboard builds cleanly.
- Phase 8 (Thought Orphanage Integration) can proceed once the live URL is established and SERAPHIM_DASHBOARD_URL is set.
- push-client.js is wired and ready — it will start sending data as soon as SERAPHIM_DASHBOARD_URL is in the environment.

---
*Phase: 07-multi-project-dashboard*
*Completed: 2026-04-08 (checkpoint — awaiting deployment)*
