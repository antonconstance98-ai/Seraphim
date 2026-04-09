# Technology Stack: v3.1 Seraphim Project Management

**Project:** Seraphim — v3.1 Project Management Layer
**Researched:** 2026-04-08
**Confidence:** HIGH (based on live system verification + GSD source reading + existing validated stack)

---

## What Is NOT Being Re-Researched

The following stack is already validated and in production. Do not change any of it:

| Technology | Version | Status |
|------------|---------|--------|
| Node.js | v22.22.0 | Installed, all plugin scripts |
| npm | 10.9.4 | Installed |
| `@openai/codex` CLI | 0.118.0 | Installed at `~/.npm-global/bin/codex` |
| `openai` npm package | 6.33.0 | Plugin executor base |
| Claude Code plugin hooks | v2.1.89+ | Active |
| Next.js + Vercel | Current | Multi-project dashboard host |
| Neon Postgres | Serverless | Dashboard persistence layer |
| Chart.js | Bundled inline | Dashboard metrics charts |

This research covers ONLY new additions required for project management features.

---

## What v3.1 Actually Needs

The goal is: roadmaps, milestones, feature queues, progress tracking, human task management, cross-project oversight, and milestone archival.

**Critical insight from studying GSD's architecture:** GSD already implements all of this with plain Markdown + JSONL files. The entire project management layer — roadmaps, milestones, requirements, state, phases, plans — lives in `.planning/` as flat files. There is no database, no ORM, no schema migration. The existing Seraphim plugin inherits this architecture.

v3.1 does not need a new persistence layer. It needs:
1. New file schemas in the existing `.planning/` structure
2. New slash commands that read/write those files
3. Dashboard panels that read and visualize the data

---

## New Stack Additions

### 1. Feature Queue: JSONL File Per Project

**File:** `.seraphim/feature-queue.jsonl`

**Why JSONL (not a new table or SQLite):** Every other event-sourced data in Seraphim is JSONL (token-log.jsonl, decision-log.jsonl). The GSD decision-log.jsonl pattern is append-only and survives crashes. JSONL integrates with the existing Node.js `fs/promises` reading pattern already in the plugin.

**Why not SQLite:** Adds a native dependency (`better-sqlite3` requires `node-gyp`, which may fail on system Node without build tools). The feature queue has fewer than a few hundred items per project — file I/O is fast enough forever.

**Why not a new Neon table:** Neon is for the dashboard read layer (cross-project aggregation for display). It is not the source of truth. The file is the source of truth; Neon is a projection of it.

```jsonl
{"id":"feat-001","title":"Roadmap view in dashboard","status":"queued","priority":"high","created":"2026-04-08","phase_target":null,"milestone":"v3.1","tags":["dashboard"]}
{"id":"feat-001","status":"in_pipeline","phase_target":"discover","updated":"2026-04-08"}
```

**No new npm packages.** Node.js built-in `fs/promises` handles append and read.

---

### 2. Progress Tracking: Extend Existing phase-state.json

**File:** `.seraphim/phases/{N}/state.json` — already exists from v3.0

**What to add to the schema:**
```json
{
  "phase": 1,
  "feature_id": "feat-001",
  "status": "in_progress",
  "started": "2026-04-08T10:00:00Z",
  "completed": null,
  "loop_counters": {},
  "checkpoint_results": []
}
```

Adding `feature_id` links a phase run to the feature that triggered it. Adding `started`/`completed` timestamps enables per-phase duration tracking. This is a schema extension to an existing file — no new files, no new packages.

---

### 3. Human Task Management: New MARKDOWN File

**File:** `.seraphim/human-tasks.md`

**Why Markdown (not JSONL):** Human tasks are things a person reads and edits — decisions to make, research to do, skills to develop. They benefit from human-readable structure. GSD's STATE.md and REQUIREMENTS.md demonstrate that Markdown is the correct format for human-facing task lists. JSONL is for machine-readable event logs.

**Format:** Standard GFM task list with metadata frontmatter:
```markdown
---
updated: 2026-04-08
---

## Decisions Needed

- [ ] [DECIDE-01] Choose primary color scheme for dashboard v2

## Research Tasks

- [ ] [RESEARCH-01] Evaluate Drizzle ORM vs raw pg for future schema needs

## Skills Development

- [ ] [SKILL-01] Learn Neon branching for staging environments
```

**Parsing:** Node.js built-in `fs/promises.readFile()` + regex on GFM task syntax. Already done in GSD tooling for plan parsing.

---

### 4. Cross-Project Overview: Neon Postgres (Existing)

The dashboard already uses Neon Postgres for cross-project metric aggregation. v3.1 adds two new tables to the existing schema.

**New tables:**

```sql
-- Feature queue snapshot (written by sync script, read by dashboard)
CREATE TABLE IF NOT EXISTS feature_snapshots (
  id TEXT NOT NULL,
  project_path TEXT NOT NULL,
  title TEXT NOT NULL,
  status TEXT NOT NULL,
  priority TEXT,
  milestone TEXT,
  tags TEXT[],
  synced_at TIMESTAMPTZ DEFAULT NOW(),
  PRIMARY KEY (id, project_path)
);

-- Human task snapshot
CREATE TABLE IF NOT EXISTS human_task_snapshots (
  id TEXT NOT NULL,
  project_path TEXT NOT NULL,
  category TEXT NOT NULL,
  title TEXT NOT NULL,
  completed BOOLEAN DEFAULT FALSE,
  synced_at TIMESTAMPTZ DEFAULT NOW(),
  PRIMARY KEY (id, project_path)
);
```

**ORM decision — raw `pg` (already installed, no addition):**

The existing dashboard sync already uses the `pg` npm package (Node-postgres) via environment variable `DATABASE_URL`. Adding two tables does not require Drizzle or Prisma. Schema changes are run as idempotent `CREATE TABLE IF NOT EXISTS` migrations inline in the sync script.

**Do not add:** Drizzle ORM, Prisma, or any migration framework. The schema is two small tables. Migration frameworks add a dev dependency, a migration directory, and a CLI step that can fail in CI. Raw `pg` with `CREATE TABLE IF NOT EXISTS` is sufficient and already in use.

**Confidence:** HIGH — `pg` package already in the dashboard project; verified from existing sync scripts.

---

### 5. Roadmap and Milestone Management: Extend Existing Files

GSD's approach (verified from source): ROADMAP.md is the human-readable roadmap. MILESTONES.md archives shipped milestones. STATE.md tracks current position. These are already present in `.planning/`.

v3.1 does not need a new roadmap file format. It needs:

1. **Milestone archival command** — `/seraphim:close-milestone` writes current roadmap snapshot to `.planning/milestones/vX.Y-ROADMAP.md` (GSD already does this pattern — see `.planning/milestones/v1.0-ROADMAP.md`)
2. **Feature-to-phase traceability** — add `feature_id` column in ROADMAP.md phase tables

No new file formats. No new packages.

---

### 6. Dashboard Panels: Next.js + React (Existing)

The v3.0 dashboard is already a Next.js app on Vercel reading from Neon. v3.1 adds panels:
- Feature queue board (Kanban-style columns: Queued → In Pipeline → Done)
- Progress per feature per phase (heatmap or table)
- Human tasks panel
- Cross-project roadmap view

**Component library decision — none (use Tailwind + existing inline styles):**

The existing dashboard uses Chart.js + Tailwind (from the v3.0 research). Adding a Kanban view does not require a drag-and-drop library — features are moved via slash commands in the terminal, not via browser drag. The dashboard is a read-heavy display layer, not an interactive editor.

**Do not add:** `@dnd-kit/core`, `react-beautiful-dnd`, `react-query`, or any state management library. The Kanban columns are static snapshots rendered from Neon query results. Server Components + `fetch` from Neon is sufficient.

**Confidence:** HIGH — architecture aligns with existing v3.0 dashboard pattern.

---

## Complete Stack for v3.1 (Delta Only)

| Addition | Type | Version | Where |
|----------|------|---------|-------|
| `feature-queue.jsonl` schema | New file format | — | `.seraphim/feature-queue.jsonl` per project |
| `human-tasks.md` schema | New file format | — | `.seraphim/human-tasks.md` per project |
| `feature_snapshots` table | New Postgres table | — | Existing Neon database |
| `human_task_snapshots` table | New Postgres table | — | Existing Neon database |
| Milestone archival pattern | New slash command | — | Extends existing GSD pattern |
| Feature queue Kanban panel | New Next.js page/component | — | Existing Vercel app |

**New npm packages required: ZERO.**

---

## What NOT to Add

| Technology | Reason |
|------------|--------|
| SQLite (`better-sqlite3`) | Requires `node-gyp` build step; unnecessary given JSONL already works for queues |
| Drizzle ORM / Prisma | Two-table schema does not justify a migration framework; raw `pg` already in use |
| `react-beautiful-dnd` / `@dnd-kit` | Kanban moves happen via terminal commands, not browser drag |
| `react-query` / `swr` | Dashboard is a server-rendered read layer; no client-side mutation needed |
| GitHub Projects API | External dependency; the project is already self-hosted with Seraphim's own management layer |
| Dedicated PM database (Supabase, PlanetScale) | Neon is already the persistence layer; don't introduce a second database provider |
| Jira/Linear/Asana integration | No requirement; Seraphim IS the project management system |
| Full-text search (Meilisearch, Typesense) | Feature lists are small enough for Postgres `ILIKE`; no dedicated search needed |

---

## Integration Points

The new data flows through the existing pipeline without changes to v3.0 components:

```
/seraphim:add-feature  →  feature-queue.jsonl (append)
/seraphim:start-feature →  dispatch.js (existing) + phase-state.json (extended)
sync-script.js          →  feature-queue.jsonl → Neon feature_snapshots
Next.js dashboard       →  SELECT from feature_snapshots → Kanban panel
```

The sync script already exists (syncs token logs and decision logs). v3.1 adds two more collection targets to the same script.

---

## Sources

- GSD source (live): `~/.claude/get-shit-done/workflows/new-project.md`, `new-milestone.md`, `execute-phase.md`
- GSD templates (live): `templates/project.md`, `templates/requirements.md`
- Seraphim project state (live): `.planning/ROADMAP.md`, `.planning/STATE.md`, `.planning/PROJECT.md`
- Seraphim v3.0 design spec (live): `docs/specs/2026-04-04-seraphim-v3-design.md`
- Node.js v22 `fs.glob()`: verified on installed v22.22.0 (prior research)
- `pg` package: in use in existing dashboard (prior research, HIGH confidence)
- Neon Postgres: https://neon.com/docs/guides/nextjs (verified as existing integration)
- AIPIM (JSONL event-log PM pattern): https://github.com/rmarsigli/aipim (MEDIUM confidence — independent confirmation of JSONL append-only approach)
- Backlog.md (markdown-native PM pattern): https://github.com/MrLesk/Backlog.md (MEDIUM confidence — confirms markdown-as-source-of-truth is a validated approach for AI-native PM)
