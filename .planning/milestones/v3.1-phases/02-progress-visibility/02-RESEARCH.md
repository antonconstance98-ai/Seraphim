# Phase 2: Progress Visibility - Research

**Researched:** 2026-04-09
**Domain:** Cross-project terminal overview, feature dependency guards, Neon PM table extension, cost aggregation
**Confidence:** HIGH

---

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions

- **D-01:** "What needs attention" is integrated into `/seraphim:overview` output as a highlighted section at the top, NOT a separate command. Surfaces: blocked features, exceeded WIP limits, pending human gates.
- **D-02:** Both auto-push via hooks AND manual `/seraphim:sync` command. Auto-push extends `phase-push.js` to detect roadmap.json and human task changes. Manual sync provides force-refresh. Same fire-and-forget pattern as v3.0.
- **D-03:** Three new tables (additive, no existing table changes): `milestones` (project, version, name, status, feature_count, complete_count, cost), `features` (project, feature_id, slug, name, status, milestone_version, pipeline_phase, cost), `human_tasks` (project, task_id, type, status, feature_id, urgency).
- **D-04:** `depends_on` array in feature schema holds feature IDs. `/seraphim:start` checks dependencies and warns (not blocks) if incomplete. Warning includes which dependencies are missing.
- **D-05:** Skills development tasks have `type: skills` with `domain` field and recommended resources. Research tasks have `type: research` with context injection — on completion, notes auto-index to project knowledge via existing RAG tools.
- **D-06:** Cross-project cost trend aggregates decisions.jsonl across all projects by date. Grouped daily. Data pushed to Neon via sync script for dashboard consumption.

### Claude's Discretion

- Exact SQL schema and column types for Neon tables
- Sync script implementation details (batch vs individual inserts)
- Overview terminal output formatting and column layout

### Deferred Ideas (OUT OF SCOPE)

None — discussion stayed within phase scope
</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| QUEUE-05 | Feature dependency declarations (`depends_on` array) with start-guard check that warns if dependencies incomplete | D-04 locked: warn-not-block; `depends_on` already in roadmap.json schema from Phase 1; start.md needs one new check block |
| TASK-05 | Skills development task type with project-domain linkage and recommended resources | D-05: `type: skills` + `domain` field; inbox.md already supports 'skills' in VALID_TYPES array; needs `domain` and `resources` fields in task marker schema |
| TASK-07 | Research task type with context injection — on completion, notes auto-index to project knowledge via RAG | D-05: `type: research` + context injection on done; done.md needs post-completion hook to index research notes |
| OVER-01 | `/seraphim:overview` shows all Seraphim projects with active milestone, features in-progress, human tasks pending, WIP count | New command; uses multi-project-scanner.js + readRoadmap per project |
| OVER-03 | Active-only filter (default) hides idle projects; `--all` flag shows everything | Argument parsing in overview.md; idle = no in-progress features AND no pending human tasks |
| OVER-04 | "What needs attention" signal surfaces blocked features, exceeded WIP limits, and pending human gates prominently | Attention section at top of overview output; rule-based (not ML) per D-01 |
| OVER-05 | Cross-project cost trend aggregating decisions.jsonl across projects by date, rendered as trend line in dashboard | D-06: daily aggregation from decisions.jsonl; synced to Neon; separate from overview terminal display |
| ARCH-04 | Neon database extended with `milestones`, `features`, `human_tasks` tables (additive, no existing table changes) | D-03: exact columns locked; only SQL types are discretionary |
| ARCH-05 | Sync script extended with two new collection targets: feature_snapshots and human_task_snapshots | Extends push-client.js and ingest route.ts; same fire-and-forget pattern |
</phase_requirements>

---

## Summary

Phase 2 builds on a solid Phase 1 foundation. All core primitives (roadmap.js, feature lifecycle, inbox, start, done) are implemented and working. This phase adds the cross-project view layer on top — a new `/seraphim:overview` command, dependency warning in `/seraphim:start`, Neon PM table extensions, and a sync mechanism that covers both roadmap and task changes.

The work divides cleanly into four tracks: (1) the overview command and attention signals, (2) the dependency guard in start.md, (3) the Neon schema additions and ingest route extension, and (4) the sync hook extension and manual sync command. Each track is independent — they can be planned and executed in parallel within a wave.

The most important architectural insight is that `multi-project-scanner.js` already does the hard work of finding all Seraphim projects. Overview only needs to call `readRoadmap()` per discovered project and aggregate the results. The Neon sync follows the exact same fire-and-forget pattern established in push-client.js — no new patterns to introduce.

**Primary recommendation:** Extend existing files in-place; introduce no new infrastructure patterns. All Phase 2 work is additive.

---

## Standard Stack

### Core (already installed, zero new packages)

| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| Node.js | v22.22.0 | Hook scripts, overview command, sync logic | Existing runtime for all plugin code |
| `@neondatabase/serverless` | installed (see db.ts) | Neon SQL client for new table DDL and upserts | Already used in dashboard; getSql() singleton |
| Next.js Route Handlers | current | Extend /api/ingest POST for PM payloads | Already the pattern for dashboard data ingestion |

### No New Packages Required

The project decision "Zero new npm packages" (STATE.md) applies to Phase 2. All required functionality is covered by:
- `fs`, `path`, `os` (Node built-ins) for file scanning
- `fetch` (Node 22 built-in) for fire-and-forget POST
- `@neondatabase/serverless` (already in dashboard package.json)

---

## Architecture Patterns

### Pattern 1: Multi-Project Scan + PM Aggregation

The overview command follows this flow:

```
discoverSeraphimProjects()          // existing: finds all .seraphim/ roots
  -> readRoadmap(projectRoot)       // existing: returns { milestones: [] } on missing
  -> countWip(roadmap)              // existing: counts in-progress features
  -> readPendingTaskCount(root)     // NEW: scan task-completions.jsonl + forge-log.md
  -> filter idle (unless --all)     // NEW: idle = wip===0 AND pending===0
  -> buildAttentionSignals(results) // NEW: rule-based blocked/wip/gate signals
  -> render terminal output         // NEW: formatted table
```

**What "idle" means:** A project is idle when it has zero in-progress features AND zero pending human tasks. `--all` flag bypasses the idle filter.

### Pattern 2: Dependency Guard in start.md

Insert between the existing WIP check and the status update. The check is purely in-process (no new files):

```javascript
// After WIP check, before feature.status = 'in-progress'
const deps = feature.depends_on || [];
const incompleteDeps = deps.filter(depId => {
  const found = r.findFeature(roadmap, depId);
  return !found || found.feature.status !== 'complete';
});
if (incompleteDeps.length > 0) {
  console.log('DEPS_WARN:' + incompleteDeps.join(','));
}
// Continue regardless (D-04: warn, not block)
```

Output line `DEPS_WARN:feat-001,feat-002` triggers a warning banner before the success message.

### Pattern 3: Neon PM Sync — Extend push-client.js

The existing `pushProjectData()` sends `decisions` + `phase_state`. The PM sync extension adds a parallel path:

```javascript
// In push-client.js — new exported function
function pushPmData(projectRoot) {
  // Read roadmap.json
  // Read task-completions.jsonl + forge-log.md for pending tasks
  // Build payload: { milestones: [...], features: [...], human_tasks: [...] }
  // POST to /api/ingest with same Bearer auth
  // fire-and-forget — never throws
}
```

Auto-trigger: extend `phase-push.js` `isPhaseOutput` check to also match `roadmap.json` and `task-completions.jsonl` writes.

Manual trigger: new `/seraphim:sync` command calls `pushPmData()` for the current project root.

### Pattern 4: Cost Trend Aggregation

Cross-project cost trend is read-only aggregation from existing decisions.jsonl files. No new data format needed:

```javascript
// In multi-project-scanner.js or a new lib/cost-aggregator.js
function aggregateCostByDate(projectRoots) {
  const byDate = {};
  for (const root of projectRoots) {
    // Read ALL decisions.jsonl (not just last 200 — trend needs full history)
    // Group by date portion of timestamp
    // Accumulate cost_usd per date
  }
  return Object.entries(byDate).map(([date, cost]) => ({ date, cost }));
}
```

This data goes into the Neon sync payload as `cost_trend` array, handled by a new ingest route branch.

### Pattern 5: Ingest Route Extension

The existing `/api/ingest` POST handler adds PM handling branches. Decision (Claude's discretion): use a `type` discriminator field in the payload to route:

```typescript
// Extend IngestPayload type
type IngestPayload =
  | { type: 'phase'; project_name: string; phase_id: string; decisions: ...; phase_state: ... }
  | { type: 'pm_snapshot'; project_name: string; milestones: ...; features: ...; human_tasks: ...; }
  | { type: 'cost_trend'; entries: { date: string; cost_usd: number; project_name: string }[] }
```

Or alternatively, extend the existing single payload shape to include optional PM fields — simpler and avoids route branching. Recommendation: extend the single payload (less refactoring, backward compatible).

### Anti-Patterns to Avoid

- **Reading all decisions.jsonl into memory for trend:** Cap to last 365 days or 10,000 lines. Unbounded reads will slow the sync for old projects.
- **Blocking pipeline on sync failures:** The PM sync must stay fire-and-forget. Network errors should log to push-errors.log and never propagate.
- **Scanning all projects for every overview render:** Cache scan results in-memory for the duration of the command (single Node process, no persistence needed).
- **Adding `depends_on` to roadmap.json schema documentation only:** Phase 1 already put `depends_on` in the D-02 schema. Verify it is actually written when `add-feature` creates a feature — if not, the planner must add a default `depends_on: []` to the add-feature write path.

---

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Project discovery | Custom glob walker | `discoverSeraphimProjects()` (multi-project-scanner.js) | Already handles ~/projects and ~/agent roots, skips non-Seraphim dirs |
| Roadmap reading | Direct fs.readFileSync in overview | `readRoadmap()` (roadmap.js) | Handles missing file, parse errors, returns safe default |
| WIP counting | Inline feature filter | `countWip()` (roadmap.js) | Already tested, handles edge cases |
| Neon connection | New neon() call | `getSql()` (dashboard/lib/db.ts) | Lazy singleton, avoids multiple connections |
| Fire-and-forget POST | Inline fetch with catch | `pushProjectData()` pattern (push-client.js) | Error logging, never throws — replicate exactly |
| Task pending count | Re-implementing inbox logic | Extract shared fn from inbox.md scan logic | Inbox already has correct task-completions.jsonl + forge-log.md logic |

**Key insight:** Phase 1 built the primitives precisely so Phase 2 could compose them. Resist reimplementing anything already in roadmap.js or multi-project-scanner.js.

---

## SQL Schema for New Neon Tables (D-03)

Columns locked by D-03. Types are Claude's discretion:

```sql
-- milestones table
CREATE TABLE IF NOT EXISTS milestones (
  id             SERIAL PRIMARY KEY,
  project_name   TEXT NOT NULL,
  version        TEXT NOT NULL,
  name           TEXT NOT NULL,
  status         TEXT NOT NULL,          -- planned | in-progress | complete | blocked
  feature_count  INTEGER NOT NULL DEFAULT 0,
  complete_count INTEGER NOT NULL DEFAULT 0,
  cost_usd       NUMERIC(10, 4) NOT NULL DEFAULT 0,
  synced_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  UNIQUE (project_name, version)
);

-- features table
CREATE TABLE IF NOT EXISTS features (
  id               SERIAL PRIMARY KEY,
  project_name     TEXT NOT NULL,
  feature_id       TEXT NOT NULL,         -- feat-NNN
  slug             TEXT,
  name             TEXT NOT NULL,
  status           TEXT NOT NULL,         -- planned | in-progress | complete | blocked
  milestone_version TEXT NOT NULL,
  pipeline_phase   TEXT,                  -- current phase if in-progress (forge, envision, etc.)
  cost_usd         NUMERIC(10, 4) NOT NULL DEFAULT 0,
  synced_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  UNIQUE (project_name, feature_id)
);

-- human_tasks table
CREATE TABLE IF NOT EXISTS human_tasks (
  id           SERIAL PRIMARY KEY,
  project_name TEXT NOT NULL,
  task_id      TEXT NOT NULL,
  type         TEXT NOT NULL,            -- decision | research | review | validation | skills
  status       TEXT NOT NULL,            -- pending | complete
  feature_id   TEXT,                     -- nullable; links to features.feature_id
  urgency      TEXT NOT NULL DEFAULT 'normal',
  synced_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  UNIQUE (project_name, task_id)
);
```

All three tables use `UNIQUE (project_name, ...)` for upsert compatibility with `ON CONFLICT DO UPDATE`.

The DDL should be run once as a migration. The planner should include a Wave 0 task to apply DDL to the Neon database before any sync tasks run.

---

## Overview Terminal Output Design

The locked constraint (D-01) is that attention signals appear at the top. Claude's discretion covers the exact layout. Recommended approach:

```
═══════════════════════════════════════════════════
 SERAPHIM  Overview
═══════════════════════════════════════════════════

 NEEDS ATTENTION
 ─────────────────────────────────────────────────
 [!] seraphim        feat-002 (auth) — blocked (depends on feat-001)
 [!] claude-x-codex  WIP limit exceeded (3/2)
 [!] agent-core      2 pending human gates
 ─────────────────────────────────────────────────

 PROJECT              MILESTONE          WIP   TASKS
 seraphim             v3.1 (60%)         1/2   3
 claude-x-codex       v1.0 (20%)         3/2!  1
 ─────────────────────────────────────────────────
 Idle projects hidden. Run /seraphim:overview --all to show all.
```

**Idle project definition:** `wip === 0 AND pendingTasks === 0`. Projects with a milestone but no activity are idle.

---

## Common Pitfalls

### Pitfall 1: `depends_on` Not Written by add-feature
**What goes wrong:** Overview and start.md read `feature.depends_on`, but if add-feature never writes the field, all features have `undefined` — `||[]` mask hides the bug until someone manually adds a dependency.
**How to avoid:** Verify add-feature.md writes `depends_on: []` as default when creating a feature. If not, the planner must add that to the add-feature implementation task.
**Warning signs:** `feature.depends_on` is undefined in existing roadmap.json files.

### Pitfall 2: Cost Trend Reading Only Last 200 Lines
**What goes wrong:** `readProjectMeta()` in multi-project-scanner.js deliberately reads only the last 200 lines for performance. Cost trend needs the full history.
**How to avoid:** The cost aggregator function must NOT use `readProjectMeta()` for trend data. It should read the full decisions.jsonl, then group by date. Cap at last 365 days.
**Warning signs:** Trend shows flat line for active projects with long history.

### Pitfall 3: Sync Trigger Fires on Every roadmap.json Rewrite
**What goes wrong:** phase-push.js extends to match `roadmap.json` writes. But roadmap.json is written on every `/seraphim:start`, `/seraphim:add-feature`, etc. — potentially many syncs per session.
**How to avoid:** Fire-and-forget is acceptable here (same as phase pushes). Network calls are non-blocking. Accept the extra pushes — each is idempotent (upsert ON CONFLICT).
**Warning signs:** push-errors.log filling up rapidly — indicates network issues, not a design flaw.

### Pitfall 4: Overview Hangs on Slow Project Reads
**What goes wrong:** If ~/projects has many subdirectories (non-Seraphim), the scanner iterates all of them. `discoverSeraphimProjects()` already filters to `.seraphim/`-containing dirs, but reading roadmaps for 10+ projects adds up.
**How to avoid:** The current synchronous read pattern is acceptable for <20 projects. If needed, the scan completes in <500ms at this scale. No async needed.

### Pitfall 5: Ingest Route Breaking Existing Callers
**What goes wrong:** Extending the IngestPayload type with required PM fields would break the existing phase push callers that don't send PM data.
**How to avoid:** All new PM fields in the ingest payload must be optional. The route handler checks `if (payload.milestones)` before running milestone upserts. Existing phase push continues unchanged.

---

## Code Examples

### Dependency Check (start.md extension)
```javascript
// Source: Phase 1 start.md pattern — inline node script
const deps = feature.depends_on || [];
const incompleteDeps = deps.filter(depId => {
  const found = r.findFeature(roadmap, depId);
  if (!found) return true;  // dep not found = treat as incomplete
  return found.feature.status !== 'complete';
});
if (incompleteDeps.length > 0) {
  const names = incompleteDeps.map(id => {
    const f = r.findFeature(roadmap, id);
    return f ? `${id} (${f.feature.name})` : id;
  });
  console.log('DEPS_WARN:' + names.join(' | '));
}
```

### Neon Upsert Pattern (from existing ingest/route.ts)
```typescript
// Source: existing dashboard/app/api/ingest/route.ts — established pattern
await sql`
  INSERT INTO milestones (project_name, version, name, status, feature_count, complete_count, cost_usd)
  VALUES (${projectName}, ${m.version}, ${m.name}, ${m.status}, ${m.feature_count}, ${m.complete_count}, ${m.cost_usd})
  ON CONFLICT (project_name, version) DO UPDATE
  SET name = EXCLUDED.name,
      status = EXCLUDED.status,
      feature_count = EXCLUDED.feature_count,
      complete_count = EXCLUDED.complete_count,
      cost_usd = EXCLUDED.cost_usd,
      synced_at = NOW()
`;
```

### Daily Cost Aggregation
```javascript
// Source: roadmap.js / multi-project-scanner.js patterns — compose existing readers
function aggregateCostByDate(projectRoots) {
  const byDate = {};
  for (const root of projectRoots) {
    const decisionsPath = path.join(root, '.seraphim', 'decisions.jsonl');
    if (!fs.existsSync(decisionsPath)) continue;
    const lines = fs.readFileSync(decisionsPath, 'utf8').split('\n').filter(Boolean);
    const projectName = path.basename(root);
    for (const line of lines) {
      try {
        const r = JSON.parse(line);
        if (typeof r.cost_usd !== 'number' || !r.timestamp) continue;
        const date = r.timestamp.slice(0, 10);  // YYYY-MM-DD
        const key = `${date}::${projectName}`;
        byDate[key] = (byDate[key] || { date, project_name: projectName, cost_usd: 0 });
        byDate[key].cost_usd += r.cost_usd;
      } catch (e) {}
    }
  }
  return Object.values(byDate);
}
```

---

## Integration Points Summary

| File | Change Type | What Changes |
|------|-------------|--------------|
| `commands/overview.md` | NEW | Full new command: scan projects, render attention signals + project table |
| `commands/sync.md` | NEW | Manual sync command: calls pushPmData for current project |
| `commands/start.md` | EXTEND | Add dependency check block between WIP check and status update |
| `commands/done.md` | EXTEND | After marking task complete, if type=research trigger RAG index note |
| `lib/multi-project-scanner.js` | EXTEND | Add `readPmSummary(projectRoot)` — returns active milestone, wip, pending tasks |
| `lib/push-client.js` | EXTEND | Add `pushPmData(projectRoot)` — reads roadmap + tasks, POSTs PM payload |
| `hooks/phase-push.js` | EXTEND | Extend `isPhaseOutput` to also match `roadmap.json` and `task-completions.jsonl` |
| `dashboard/app/api/ingest/route.ts` | EXTEND | Add PM branches: milestones, features, human_tasks, cost_trend upserts |
| `dashboard/lib/types.ts` | EXTEND | Add PM fields to IngestPayload (all optional) |
| Neon database | MIGRATE | Apply DDL for 3 new tables (Wave 0 task) |

---

## Environment Availability

| Dependency | Required By | Available | Version | Fallback |
|------------|------------|-----------|---------|----------|
| Node.js | All plugin scripts | Yes | v22.22.0 | — |
| Neon database (DATABASE_URL) | ARCH-04, ARCH-05 | Assumed yes (existing dashboard uses it) | — | Log to push-errors.log; sync is fire-and-forget |
| SERAPHIM_DASHBOARD_URL env var | Auto-sync push | Set if auto-sync is active | — | Sync silently skipped (same as existing push-client.js behavior) |
| SERAPHIM_DASHBOARD_SECRET env var | Auth on ingest | Set with dashboard URL | — | Empty string allowed (ingest route checks for non-empty expected token) |

---

## Open Questions

1. **Does add-feature.md write `depends_on: []` by default?**
   - What we know: D-02 schema includes `depends_on` in the spec; add-feature.md was created in Phase 1
   - What's unclear: Whether the actual write in add-feature.md includes the field or omits it
   - Recommendation: Planner must include a verification step in the dependency guard task — read the current add-feature.md and confirm or add the default field

2. **Does `done.md` currently have a mechanism to detect task type?**
   - What we know: done.md marks tasks complete by writing to task-completions.jsonl
   - What's unclear: Whether it reads the task's type field to know if it's `research` type
   - Recommendation: The RAG index integration (TASK-07) requires done.md to read the task record before completing — planner should check current done.md and extend accordingly

3. **Where does the Neon DDL migration get applied?**
   - What we know: No migration runner exists; existing tables were created manually
   - Recommendation: Planner should include a Wave 0 task with explicit SQL to run against Neon, with instructions for the human to run it via Neon console or psql

---

## Sources

### Primary (HIGH confidence)
- `~/.claude/plugins/seraphim/lib/roadmap.js` — confirmed API: readRoadmap, writeRoadmap, findFeature, nextFeatureId, countWip, milestoneProgress
- `~/.claude/plugins/seraphim/lib/multi-project-scanner.js` — confirmed API: discoverSeraphimProjects, readProjectMeta
- `~/.claude/plugins/seraphim/lib/push-client.js` — confirmed fire-and-forget pattern, push-errors.log location
- `~/.claude/plugins/seraphim/hooks/phase-push.js` — confirmed trigger condition, isPhaseOutput regex
- `~/.claude/plugins/seraphim/dashboard/app/api/ingest/route.ts` — confirmed upsert pattern, auth, payload shape
- `~/.claude/plugins/seraphim/dashboard/lib/db.ts` — confirmed getSql() singleton pattern
- `~/.claude/plugins/seraphim/commands/start.md` — confirmed WIP check flow, output protocol
- `~/.claude/plugins/seraphim/commands/inbox.md` — confirmed task scan pattern, VALID_TYPES list
- `.planning/phases/02-progress-visibility/02-CONTEXT.md` — all locked decisions (D-01 through D-06)
- `.planning/REQUIREMENTS.md` — all phase 2 requirement definitions

### Secondary (MEDIUM confidence)
- `.planning/STATE.md` — "Zero new npm packages" decision confirmed
- `.planning/phases/01-core-pm-primitives/01-CONTEXT.md` — D-02 roadmap.json schema (depends_on in spec)

---

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH — confirmed from installed files; zero new packages required
- Architecture patterns: HIGH — all patterns are extensions of verified Phase 1 code
- Pitfalls: HIGH — derived from reading actual code (last-200-lines cap in scanner, isPhaseOutput regex, existing payload shape)
- SQL schema: MEDIUM — column names locked by D-03; types are reasonable defaults but not verified against existing Neon schema

**Research date:** 2026-04-09
**Valid until:** 2026-05-09 (stable domain — plugin code won't change between research and planning)
