# Phase 32: Foundations - Research

**Researched:** 2026-04-09
**Domain:** Neon DDL migrations, schema unification, feature_id wiring, schema extension audit
**Confidence:** HIGH — all findings verified directly from live codebase files

---

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions
- **D-01:** Migration SQL files committed to repo at `dashboard/migrations/`. Each migration is idempotent (`CREATE TABLE IF NOT EXISTS`, `ALTER TABLE ... ADD COLUMN IF NOT EXISTS`).
- **D-02:** Migrations applied via a `migrate.ts` script that runs on deploy or manually via `npx tsx dashboard/migrations/migrate.ts`. No ORM — raw SQL through @neondatabase/serverless.
- **D-03:** PM tables needed: `projects`, `milestones`, `features`, `human_tasks`, `decisions`, `cost_snapshots`. These match the push-client.js payload structure. If tables already exist partially, alter to match.
- **D-04:** Standardize on `project_name` (snake_case string) as the canonical project identifier across ALL Neon tables and all push-client payloads.
- **D-05:** Any existing columns using `project` (without `_name`) get an `ALTER TABLE RENAME COLUMN` in the migration. Dashboard queries updated to match.
- **D-06:** In-code JavaScript uses camelCase `projectName` per JS conventions. Snake_case `project_name` only in SQL and JSON payloads.
- **D-07:** Trace all callers of `decisions-logger.js` buildRecord(). Any caller that has access to the current feature context must pass `feature_id`. Callers without feature context continue passing `null`.
- **D-08:** The feature_id comes from `.seraphim/roadmap.json` active feature. The `pm-context.js` module already exposes this — wire it through to decisions-logger at the call sites.
- **D-09:** Audit document written to `.planning/phases/32-foundations/SCHEMA-AUDIT.md`. Maps each v3.2 data concept to its home in existing structures.
- **D-10:** The audit confirms "extend, not duplicate" for every concept. If a concept requires a new top-level data store, it must justify why the existing store can't be extended.

### Claude's Discretion
- All four gray areas (Neon DDL strategy, schema unification, extension audit format, feature_id wiring) — user gave full discretion.

### Deferred Ideas (OUT OF SCOPE)
- None — discussion stayed within phase scope
</user_constraints>

---

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| FOUND-01 | v3.1 Neon DDL applied — all PM tables exist in production Neon | DDL analysis confirms zero migrations dir exists; tables referenced in ingest route but never created via committed SQL. Migration script + DDL files are the deliverable. |
| FOUND-02 | Schema consistency — `project` vs `project_name` mismatch resolved across all tables | Direct observation: queries.ts uses `WHERE project = ${projectName}` for milestones/features/human_tasks/cost_trend, but ingest route uses `project_name` for projects/decisions tables. Four queries need column rename + query update. |
| FOUND-03 | `feature_id` flows through decisions-logger to Neon | Two-gap problem: (1) ingest INSERT into decisions omits feature_id column entirely; (2) crucible.md and judge.md pass `phaseId` (pipeline phase string like "crucible") as feature_id instead of the actual feat-NNN ID from roadmap.json. |
| FOUND-04 | Schema extension audit — every v3.2 data concept extends existing structures | Audit document maps 7 v3.2 data concepts to existing structures. All can extend without new top-level stores. |
</phase_requirements>

---

## Summary

Phase 32 is a technical debt clearance gate before any v3.2 feature work starts. Three concrete code problems were identified by reading the live codebase directly, and one documentation artifact must be produced.

**FOUND-01 (Neon DDL):** The `dashboard/migrations/` directory does not exist. The ingest route at `dashboard/app/api/ingest/route.ts` issues INSERT statements against `projects`, `decisions`, `phase_states`, `milestones`, `features`, `human_tasks`, and `cost_trend` tables — but there is no committed DDL that creates them. The migration script and SQL files must be created from scratch by reading the ingest route to derive the expected schema.

**FOUND-02 (Schema unification):** The mismatch is fully observable. `queries.ts` uses `project` as the column name in WHERE clauses for four PM tables (milestones, features, human_tasks, cost_trend). The ingest route inserts into those same tables using the payload field `project_name`. This means ingest succeeds but queries return zero rows. Fix: one ALTER TABLE RENAME COLUMN per affected table, plus update the four query functions in queries.ts.

**FOUND-03 (feature_id wiring):** Two independent gaps exist. First, the ingest route's INSERT into `decisions` omits `feature_id` from the column list and values entirely — the column either doesn't exist in the live schema or is never written. Second, `crucible.md` line 336 and `judge.md` line 255 both pass `feature_id: phaseId` where `phaseId` is a pipeline phase string ("crucible", "forge") not a feat-NNN roadmap ID. The correct source is `readRoadmap(projectRoot)` to find the active feature (status === 'in-progress'), then extract its `id`. Note: no `pm-context.js` module was found at the expected path — the roadmap module (`roadmap.js`) exposes `readRoadmap` and `findFeature` and is the correct source.

**FOUND-04 (Schema extension audit):** The audit maps 7 v3.2 data concepts. All can extend existing structures without new top-level data stores. The audit document is a write-only task — no code change required.

**Primary recommendation:** Four sequential tasks: (1) create migrations dir + DDL + migrate.ts; (2) rename `project` column to `project_name` in PM tables + update queries.ts; (3) add `feature_id` to decisions INSERT in ingest route + fix feature_id source in crucible.md and judge.md; (4) write SCHEMA-AUDIT.md.

---

## Standard Stack

### Core (already installed — zero new dependencies per v3.2 constraint)

| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| @neondatabase/serverless | existing (dashboard/package.json) | Neon SQL client — tagged template literal queries | Already used in db.ts and ingest route; no additional driver needed |
| tsx | existing | Run TypeScript scripts directly via `npx tsx` | Used by convention in Next.js projects; already available |
| Node.js fs module | built-in | Read roadmap.json for active feature_id | Already used by roadmap.js; no new deps |

**Installation:** No new packages required.

### The SQL pattern in use

```typescript
// From dashboard/lib/db.ts — getSql() returns a tagged template literal function
const sql = getSql();
await sql`INSERT INTO decisions (...) VALUES (...)`;
await sql`ALTER TABLE milestones RENAME COLUMN project TO project_name`;
```

---

## Architecture Patterns

### Migration File Structure (to create)

```
dashboard/
├── migrations/
│   ├── 001-initial-pm-schema.sql   # CREATE TABLE IF NOT EXISTS for all 7 tables
│   └── migrate.ts                  # Reads and applies SQL files via getSql()
```

### Pattern 1: Idempotent DDL (per D-01)

**What:** Every CREATE and ALTER uses existence guards so re-running is safe.

```sql
-- Create tables
CREATE TABLE IF NOT EXISTS projects (
  name TEXT PRIMARY KEY,
  root_path TEXT,
  profile TEXT,
  last_pushed_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE IF NOT EXISTS decisions (
  id SERIAL PRIMARY KEY,
  project_name TEXT NOT NULL,
  timestamp TIMESTAMPTZ,
  phase TEXT,
  model TEXT,
  profile TEXT,
  tokens_in INT DEFAULT 0,
  tokens_out INT DEFAULT 0,
  cost_usd NUMERIC DEFAULT 0,
  latency_ms INT DEFAULT 0,
  outcome TEXT,
  retry_count INT DEFAULT 0,
  loop_count INT DEFAULT 0,
  feature_id TEXT,          -- required for FOUND-03
  quality_signals JSONB DEFAULT '{}'
);

CREATE TABLE IF NOT EXISTS phase_states (
  id SERIAL PRIMARY KEY,
  project_name TEXT NOT NULL,
  phase_id TEXT NOT NULL,
  completed BOOLEAN DEFAULT FALSE,
  completed_at TIMESTAMPTZ,
  loops JSONB DEFAULT '{}',
  retries JSONB DEFAULT '{}',
  updated_at TIMESTAMPTZ DEFAULT NOW(),
  UNIQUE(project_name, phase_id)
);

-- PM tables — use project_name (unified per D-04)
CREATE TABLE IF NOT EXISTS milestones (
  id SERIAL PRIMARY KEY,
  project_name TEXT NOT NULL,    -- was "project" in old queries
  version TEXT NOT NULL,
  name TEXT,
  status TEXT,
  progress INT DEFAULT 0,
  cost_usd NUMERIC DEFAULT 0,
  synced_at TIMESTAMPTZ DEFAULT NOW(),
  UNIQUE(project_name, version)
);

CREATE TABLE IF NOT EXISTS features (
  id SERIAL PRIMARY KEY,
  project_name TEXT NOT NULL,    -- was "project" in old queries
  feature_id TEXT NOT NULL,
  slug TEXT,
  name TEXT,
  status TEXT,
  milestone_version TEXT,
  pipeline_phase TEXT,
  cost_usd NUMERIC DEFAULT 0,
  synced_at TIMESTAMPTZ DEFAULT NOW(),
  UNIQUE(project_name, feature_id)
);

CREATE TABLE IF NOT EXISTS human_tasks (
  id SERIAL PRIMARY KEY,
  project_name TEXT NOT NULL,    -- was "project" in old queries
  task_id TEXT NOT NULL,
  type TEXT,
  status TEXT,
  feature_id TEXT,
  urgency TEXT,
  synced_at TIMESTAMPTZ DEFAULT NOW(),
  UNIQUE(project_name, task_id)
);

CREATE TABLE IF NOT EXISTS cost_trend (
  id SERIAL PRIMARY KEY,
  project_name TEXT NOT NULL,    -- was "project" in old queries
  date DATE NOT NULL,
  cost_usd NUMERIC DEFAULT 0,
  synced_at TIMESTAMPTZ DEFAULT NOW(),
  UNIQUE(project_name, date)
);
```

**Note on existing live schema:** If the tables were manually created before this phase with the old `project` column name, an ALTER TABLE RENAME COLUMN is needed in the migration:

```sql
-- Run only if tables already exist with old column name
-- Safe to include with IF EXISTS guard
ALTER TABLE milestones RENAME COLUMN project TO project_name;
ALTER TABLE features RENAME COLUMN project TO project_name;
ALTER TABLE human_tasks RENAME COLUMN project TO project_name;
ALTER TABLE cost_trend RENAME COLUMN project TO project_name;
```

The migration script should attempt both approaches (CREATE IF NOT EXISTS + conditional RENAME) and catch errors gracefully.

### Pattern 2: migrate.ts Script (per D-02)

```typescript
// dashboard/migrations/migrate.ts
import { getSql } from '../lib/db';
import { readFileSync } from 'fs';
import { join } from 'path';

async function migrate() {
  const sql = getSql();
  const migrationFile = join(__dirname, '001-initial-pm-schema.sql');
  const ddl = readFileSync(migrationFile, 'utf8');
  // Execute as a single batch
  await sql.query(ddl);
  console.log('Migration complete');
}

migrate().catch(console.error);
```

**Run with:** `npx tsx dashboard/migrations/migrate.ts`

### Pattern 3: feature_id Source (per D-08)

The correct way to get the active feature_id from a command context:

```javascript
// In crucible.md / judge.md logging block
const { readRoadmap, findFeature } = require(`${CLAUDE_PLUGIN_ROOT}/lib/roadmap.js`);
const roadmap = readRoadmap(projectRoot);
// Find the active feature (in-progress status)
let activeFeatureId = null;
if (roadmap && roadmap.milestones) {
  for (const milestone of roadmap.milestones) {
    for (const feature of (milestone.features || [])) {
      if (feature.status === 'in-progress') {
        activeFeatureId = feature.id;  // e.g. "feat-003"
        break;
      }
    }
    if (activeFeatureId) break;
  }
}

const record = buildRecord({
  // ... other fields ...
  feature_id: activeFeatureId,  // feat-NNN or null, NOT phaseId
});
```

**Note:** No `pm-context.js` file exists at the plugin lib path. `roadmap.js` is the correct source. The `pm-context.js` reference in CONTEXT.md refers to a module that either was planned but not yet created, or exists elsewhere. The roadmap module's `readRoadmap()` + inline active feature search is the correct approach.

### Pattern 4: feature_id in Ingest Route

The ingest route INSERT into `decisions` must be extended:

```typescript
// dashboard/app/api/ingest/route.ts — decisions INSERT
await sql`
  INSERT INTO decisions (
    project_name, timestamp, phase, model, profile,
    tokens_in, tokens_out, cost_usd, latency_ms, outcome,
    retry_count, loop_count, quality_signals, feature_id   -- ADD feature_id
  ) VALUES (
    ${payload.project_name},
    ${d.timestamp},
    ${d.phase},
    ${d.model},
    ${d.profile ?? null},
    ${d.tokens_in ?? 0},
    ${d.tokens_out ?? 0},
    ${d.cost_usd ?? 0},
    ${d.latency_ms ?? 0},
    ${d.outcome ?? null},
    ${d.retry_count ?? 0},
    ${d.loop_count ?? 0},
    ${JSON.stringify(d.quality_signals ?? {})},
    ${(d as any).feature_id ?? null}                        -- ADD feature_id value
  )
`;
```

The `Decision` type in `types.ts` must also add `feature_id?: string | null`.

### Pattern 5: queries.ts Column Name Updates (per D-05)

Four query functions need WHERE clause updates after column rename:

```typescript
// Before (wrong):  WHERE project = ${projectName}
// After (correct): WHERE project_name = ${projectName}

// Affected functions in queries.ts:
// - getMilestones()         line 57-65
// - getFeatures()           line 67-76
// - getHumanTasks()         line 78-87
// - getProjectCostTrend()   line 89-99
```

### Anti-Patterns to Avoid

- **Creating pm-context.js from scratch:** roadmap.js already provides everything needed. Adding a new module would duplicate the roadmap read logic.
- **Using `project` as the column name in new DDL:** The decision is standardized on `project_name`. Never use bare `project` in new SQL.
- **Blocking ingest on migration failure:** The migration is a one-time setup step; the ingest route should not run the migration at request time.
- **Passing pipeline phase string as feature_id:** `phaseId` is "crucible" or "forge", not a feat-NNN. These are different concepts. The roadmap's in-progress feature is the correct source.

---

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| SQL schema management | Custom ORM or schema builder | Raw SQL with `CREATE TABLE IF NOT EXISTS` | D-02 explicitly prohibits ORM; pattern already established in codebase |
| Active feature lookup | New pm-context.js module | `readRoadmap()` from roadmap.js + inline loop | roadmap.js already exists and exports readRoadmap; adding a new file is unnecessary complexity |
| Neon connection | New connection factory | `getSql()` from dashboard/lib/db.ts | Singleton pattern already implemented; importing getSql is the established pattern |
| TypeScript script runner | Compile + node | `npx tsx` | Already available; zero additional setup |

---

## Common Pitfalls

### Pitfall 1: ALTER TABLE on a Table That Doesn't Exist Yet

**What goes wrong:** The migration tries `ALTER TABLE milestones RENAME COLUMN project TO project_name` but the table was never created (DDL was pending). ALTER fails, migration aborts.

**Why it happens:** The migration assumes the table exists from a prior manual step that may not have happened.

**How to avoid:** In the migration, `CREATE TABLE IF NOT EXISTS` first (with `project_name` as the column name), then attempt the RENAME only if the old column exists. Check via `information_schema.columns` before issuing the RENAME, or wrap in a PL/pgSQL DO block with exception handling.

**Warning signs:** Migration script errors with "table does not exist" or "column does not exist".

### Pitfall 2: ingest Route Fails Silently After Column Rename

**What goes wrong:** After renaming `project` to `project_name`, the ingest route INSERT for milestones/features/human_tasks/cost_trend still references the old column name and fails with a DB error. But the ingest route only returns `{ ok: true }` on the projects/decisions/phase_states upserts — PM data silently drops.

**Why it happens:** The ingest route INSERT statements reference column names directly. Renaming a column does not auto-update query strings.

**How to avoid:** After applying the migration, search the ingest route for `INSERT INTO milestones`, `INSERT INTO features`, `INSERT INTO human_tasks`, `INSERT INTO cost_trend` and verify each column list uses `project_name`. They currently use `project` (line 85, 98, 113, 124 of route.ts).

**Warning signs:** Dashboard shows projects and decisions but zero milestones/features after the migration.

### Pitfall 3: feature_id Has Wrong Value Even After Wiring

**What goes wrong:** The logging block in crucible.md passes `feature_id: phaseId` where `phaseId` is "crucible". After the fix, the code reads roadmap but the active feature has a different status value (e.g., "active" instead of "in-progress").

**How to avoid:** Confirm the actual status value used by the Seraphim pipeline by reading a real `.seraphim/roadmap.json` file. The roadmap.js `countWip` function checks `status === 'in-progress'` (line 89), which confirms "in-progress" is the canonical active status.

**Warning signs:** `feature_id` is null in all decisions after the fix.

### Pitfall 4: Decision Type Missing feature_id

**What goes wrong:** The `Decision` interface in `types.ts` does not include `feature_id`. TypeScript will error when the ingest route tries to access `d.feature_id`. Since the ingest route casts to `IngestPayload`, and `Decision` omits the field, `d.feature_id` silently resolves to `undefined`.

**How to avoid:** Add `feature_id?: string | null` to the `Decision` interface in `types.ts` before updating the ingest route.

### Pitfall 5: migrate.ts Cannot Access DATABASE_URL

**What goes wrong:** `npx tsx dashboard/migrations/migrate.ts` runs in the shell but `process.env.DATABASE_URL` is undefined because the Neon connection string is in `.env.local` (not loaded by tsx by default).

**How to avoid:** Either: (a) run with `npx dotenv -e .env.local -- tsx dashboard/migrations/migrate.ts`, or (b) have the migration script accept DATABASE_URL as a CLI argument, or (c) document that the user must export DATABASE_URL before running. The `.env.local` approach is the Next.js convention.

---

## Code Examples

### Verified: Current ingest route INSERT column lists (what needs updating)

```typescript
// route.ts line 85 — milestones INSERT (currently uses "project", must be "project_name")
INSERT INTO milestones (project, version, name, status, progress, cost_usd)

// route.ts line 98 — features INSERT (currently uses "project")
INSERT INTO features (project, feature_id, name, status, milestone_version, cost_usd)

// route.ts line 113 — human_tasks INSERT (currently uses "project")
INSERT INTO human_tasks (project, task_id, type, status, feature_id, urgency)

// route.ts line 124 — cost_trend INSERT (currently uses "project")
INSERT INTO cost_trend (project, date, cost_usd)
```

### Verified: Current decisions INSERT (missing feature_id)

```typescript
// route.ts lines 36-56 — decisions INSERT omits feature_id entirely
INSERT INTO decisions (
  project_name, timestamp, phase, model, profile,
  tokens_in, tokens_out, cost_usd, latency_ms, outcome,
  retry_count, loop_count, quality_signals
  -- feature_id is missing here and in VALUES
)
```

### Verified: Crucible/Judge wrong feature_id source

```javascript
// crucible.md line 336, judge.md line 255 — WRONG
feature_id: phaseId,   // phaseId = "crucible" or "forge" — this is a pipeline phase, not a roadmap feature ID
```

### Verified: roadmap.js active feature pattern

```javascript
// roadmap.js line 89 — canonical in-progress check
if (feature.status === 'in-progress') count++;
// Confirmed: 'in-progress' is the canonical active feature status
```

---

## Schema Extension Audit (FOUND-04 content)

The following maps each v3.2 data concept to its target structure. This section is the basis for the SCHEMA-AUDIT.md document.

| v3.2 Concept | Existing Home | Extension Approach | New Store? |
|--------------|--------------|-------------------|-----------|
| Seeds | `.planning/seeds/` directory (exists) | Add `index.jsonl` as lookup index. Each seed is a SEED-NNN.md file. | No new top-level store — directory already exists |
| Requirements | `.seraphim/roadmap.json` features array | Add `req_id`, `scope` fields to existing feature objects following D-09 | No — extends roadmap.json |
| Waves | `.seraphim/roadmap.json` milestones array | Add `waves[]` array or `wave_id` field to milestone objects | No — extends roadmap.json |
| Discuss decisions | `.planning/phases/NN-slug/NN-CONTEXT.md` | Already exists — decisions go in CONTEXT.md per current workflow | No new file |
| Research items | `.seraphim/research.json` (new file) | Follows roadmap.json atomic write pattern (`readResearch`/`writeResearch`) | Yes — justified because research items are cross-feature (not scoped to one feature) and have independent lifecycle |
| Human task enrichment | Existing HumanTask type | Add optional `skills_to_learn`, `thought_prompt`, `research_task` fields to HumanTaskSnapshot | No — extends existing type |
| Progress/velocity | `.seraphim/timeline.jsonl` + `task-completions.jsonl` | Compute from existing append-only logs | No — computed, no new store |

**Finding:** Six of seven concepts extend existing structures. `research.json` is the only new file justified by cross-feature scope. This satisfies D-10.

---

## Environment Availability

| Dependency | Required By | Available | Version | Fallback |
|------------|------------|-----------|---------|----------|
| DATABASE_URL (Neon connection string) | migrate.ts, ingest route | Unknown — env var not checkable | — | Must be set before migration; check `.env.local` |
| npx tsx | migrate.ts runner | Likely available (Next.js project has tsx) | — | `ts-node` as alternative |
| @neondatabase/serverless | DDL execution | Yes (in dashboard/package.json) | existing | — |
| SERAPHIM_DASHBOARD_URL | Verifying push works after migration | Set in env (push-client.js checks it) | — | Manual SQL query to verify |

**Missing dependencies with no fallback:**
- `DATABASE_URL` must be set in `.env.local` or shell environment before running `migrate.ts`. The planner must include a verification step confirming this env var is present before running the migration.

---

## Validation Architecture

### Test Framework

No automated test framework was found for this phase. Phase 32 changes are infrastructure-level (DDL + query updates) and are validated by running the actual queries against Neon.

| Property | Value |
|----------|-------|
| Framework | None detected for dashboard/ (Next.js project, no jest/vitest config found) |
| Quick run command | `npx tsx dashboard/migrations/migrate.ts` |
| Full suite command | Manual: run migrate.ts, then call the ingest endpoint with a test payload, then run each query function |

### Phase Requirements → Test Map

| Req ID | Behavior | Test Type | Automated Command | Manual Step |
|--------|----------|-----------|-------------------|-------------|
| FOUND-01 | All PM tables exist in Neon | smoke | `npx tsx dashboard/migrations/migrate.ts` | Verify no errors; optionally run `SELECT 1 FROM milestones LIMIT 1` via psql or dashboard |
| FOUND-02 | `project_name` column consistent | smoke | Run migrate.ts (includes RENAME), then push a test payload and verify `getMilestones()` returns data | Dashboard load without zero-row bug |
| FOUND-03 | feature_id in decisions | smoke | Run crucible or judge on a project with active feat-NNN; check `.seraphim/decisions.jsonl` for `feature_id` field with feat-NNN value | Check Neon decisions table for non-null feature_id |
| FOUND-04 | SCHEMA-AUDIT.md exists | manual | File exists at `.planning/phases/32-foundations/SCHEMA-AUDIT.md` | Review content for completeness |

### Wave 0 Gaps

- [ ] `dashboard/migrations/` directory — does not exist, must be created in Wave 1
- [ ] `dashboard/migrations/001-initial-pm-schema.sql` — the DDL file
- [ ] `dashboard/migrations/migrate.ts` — the runner script

---

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| `project` column name | `project_name` (unified) | Phase 32 | Four queries + four INSERT statements need update |
| `feature_id: phaseId` | `feature_id: activeFeatureId` from roadmap | Phase 32 | Decisions gain proper feature attribution for v3.2 dashboard queries |
| No committed DDL | `dashboard/migrations/001-initial-pm-schema.sql` | Phase 32 | Schema becomes reproducible; Neon can be reset/restored |

---

## Open Questions

1. **Are the PM tables already partially created in Neon from prior manual steps?**
   - What we know: The migrations directory doesn't exist in the repo. The ingest route has been live since v3.1.
   - What's unclear: Whether someone manually executed CREATE TABLE statements against the production Neon instance.
   - Recommendation: The migration script should handle both cases — use `CREATE TABLE IF NOT EXISTS` so it is safe to run whether tables exist or not. If the `project` column exists, attempt a RENAME; if `project_name` already exists, skip the RENAME gracefully.

2. **Does pm-context.js exist anywhere in the plugin?**
   - What we know: CONTEXT.md references it, but it was not found at `~/.claude/plugins/seraphim/lib/pm-context.js`.
   - What's unclear: Whether it exists under a different name or path, or whether it was a planned-but-unbuilt module from a prior discussion.
   - Recommendation: Do not depend on pm-context.js. Use `readRoadmap()` from roadmap.js directly at call sites in crucible.md and judge.md. If pm-context.js is needed for future phases, build it then.

3. **Does `@neondatabase/serverless` support raw SQL string execution (for DDL batches)?**
   - What we know: The `getSql()` returns a tagged template literal function. Tagged template literals work for parameterized queries.
   - What's unclear: Whether multi-statement DDL (multiple CREATE TABLE statements in one file) can be passed as a single call, or must be split per statement.
   - Recommendation: Split the DDL into individual statements in migrate.ts and execute each separately to avoid any multi-statement parsing issues. Alternatively, use `sql.query(ddl)` if the neon driver exposes a raw query method.

---

## Sources

### Primary (HIGH confidence — read directly from codebase)

- `/home/alucardmessangeroflight/.claude/plugins/seraphim/dashboard/lib/queries.ts` — confirmed `project` column name in WHERE clauses
- `/home/alucardmessangeroflight/.claude/plugins/seraphim/dashboard/app/api/ingest/route.ts` — confirmed `project` column in PM INSERT statements and missing `feature_id` in decisions INSERT
- `/home/alucardmessangeroflight/.claude/plugins/seraphim/dashboard/lib/types.ts` — confirmed Decision type lacks feature_id field
- `/home/alucardmessangeroflight/.claude/plugins/seraphim/lib/decisions-logger.js` — confirmed buildRecord() has feature_id parameter with null default
- `/home/alucardmessangeroflight/.claude/plugins/seraphim/lib/push-client.js` — confirmed project_name in payload (line 78), project in PM INSERT (lines 85, 98, 113, 124 of route)
- `/home/alucardmessangeroflight/.claude/plugins/seraphim/lib/roadmap.js` — confirmed readRoadmap pattern and 'in-progress' status string
- `/home/alucardmessangeroflight/.claude/plugins/seraphim/commands/crucible.md` lines 309, 336 — confirmed `feature_id: phaseId` bug
- `/home/alucardmessangeroflight/.claude/plugins/seraphim/commands/judge.md` lines 229, 255 — confirmed same bug
- `/home/alucardmessangeroflight/projects/seraphim/.planning/research/PITFALLS.md` — Pitfall 3 confirms Neon schema divergence is a known risk
- `.planning/phases/32-foundations/32-CONTEXT.md` — all locked decisions D-01 through D-10

### Secondary (MEDIUM confidence)

- `STATE.md` — confirms Phase 32 is the prerequisite gate, Neon DDL pending
- `REQUIREMENTS.md` — confirms FOUND-01 through FOUND-04 scope

---

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH — all libraries already in use; no new packages
- Architecture: HIGH — DDL derived directly from live ingest route column lists
- Pitfalls: HIGH — bugs identified by reading actual code, not inferred
- Schema extension audit: HIGH — all 7 concepts read from CONTEXT.md D-09, cross-checked against existing file structure

**Research date:** 2026-04-09
**Valid until:** 2026-05-09 (stable domain — Neon SQL, no fast-moving libraries)
