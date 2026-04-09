# Technology Stack: v3.2 Idea-to-Shipped Journey

**Project:** Seraphim — v3.2 Idea-to-Shipped Workflow
**Researched:** 2026-04-09
**Confidence:** HIGH (all findings based on live codebase inspection)

---

## Core Finding: Zero New Dependencies

After inspecting the full plugin stack (plugin `package.json`, dashboard `package.json`, all lib modules, existing data files), v3.2 needs no new npm packages. Every required capability is already present.

| v3.2 Feature | Capability Needed | Already Covered By |
|-------------|------------------|--------------------|
| Seed/Idea Capture | Persistent markdown + state index | Node.js `fs` — pattern proven by SEED-001 in `.planning/seeds/` |
| Research System | Categorized notes, web search | Node.js `fs` + Perplexity MCP (already configured) |
| Requirements Definition | REQ-ID generation, scoped lists | Node.js `fs` — REQUIREMENTS.md pattern already used in v3.0 |
| Phased Roadmaps with waves/deps | Wave JSON, dependency resolution | Extend `lib/roadmap.js` — inline topological sort (~40 lines) |
| Discuss Phase | Decision lock state flag | Node.js `fs`, `.seraphim/discuss-state.json` |
| Planning System | PLAN.md generation, task tracking | Node.js `fs` — existing phase system already generates markdown |
| Verification | Goal-backward check scripts | Node.js `fs` + existing Crucible phase infrastructure |
| Enriched Human Tasks | Extended task types with skills/research | Extend existing `human-tasks.md` schema (v3.1 established) |
| Progress Visualization | Completion bars, velocity | Chart.js 4.5.1 — already in dashboard, horizontal bar is built-in |
| Dashboard Control Center | Full roadmap/wave/task/cost view | Next.js 16.2.3 + @neondatabase/serverless 1.0.2 — already in place |

---

## Existing Stack Reference (No Changes)

### Plugin Runtime

| Technology | Version | Purpose |
|------------|---------|---------|
| Node.js | 22.22.0 | All command handlers, lib modules, file I/O |
| better-sqlite3 | 12.8.0 (installed) | Local SQLite for RAG — reuse for seed/research indexing |
| @huggingface/transformers | 4.0.1 (installed) | Local embeddings — seeds and research docs auto-indexed on write |
| sqlite-vec | 0.1.9 (installed) | Vector similarity search for RAG |

### Dashboard

| Technology | Version | Purpose |
|------------|---------|---------|
| Next.js | 16.2.3 | API routes + SSR for new roadmap/wave/progress pages |
| React | 19.2.4 | Dashboard UI components |
| @neondatabase/serverless | 1.0.2 | Neon Postgres queries |
| Chart.js | 4.5.1 | Progress bars (horizontal bar), velocity (line), burndown (line) |
| Tailwind CSS | 4.x | Styling — no additional component library needed |
| TypeScript | 5.x | Types for new data models |

---

## New Data Models (File Schemas — No New Packages)

### Seed Store

**Location:** `.planning/seeds/SEED-NNN-slug.md` + `.planning/seeds/index.jsonl`

The format is already established — SEED-001 exists live. The `index.jsonl` is new for fast status lookups without scanning all markdown files.

```jsonl
{"id":"SEED-001","slug":"self-improving-intelligence","status":"dormant","planted":"2026-04-03","scope":"Large"}
```

**On write:** Call `lib/rag-indexer.js` to index the seed into local SQLite. Seeds become searchable context in future Discover phases. The `reindex` command already handles batch reindex if the index falls out of sync.

### Research Store

**Location:** `.planning/research/` (this directory — already used for STACK.md, FEATURES.md, etc.)

New: `.planning/research/index.jsonl` tracking research items by status:

```jsonl
{"id":"RES-001","question":"What rate limits apply to Gemini 3.1 Pro?","category":"api","status":"open","created":"2026-04-09"}
{"id":"RES-001","status":"answered","answer":"1000 RPM on free tier","source":"https://ai.google.dev/pricing","updated":"2026-04-09"}
```

Research docs are append-only events (same pattern as `decision-log.jsonl`). The dashboard reads the latest state per ID.

### Requirements Store

**Location:** `.planning/REQUIREMENTS.md` (reinstate — was deleted during v3.1 audit) + `.seraphim/requirements.json` (machine-readable)

The `.seraphim/requirements.json` enables the roadmap validator to check that all `v1` requirements map to at least one feature:

```json
{
  "requirements": [
    { "id": "REQ-001", "text": "Seed capture via /seed command", "scope": "v1", "feature_ids": ["feat-012"] }
  ]
}
```

### Phased Roadmap with Waves + Dependencies

Extend the existing `roadmap.json` schema. Current structure is flat (`milestones → features`). v3.2 adds a wave layer and dependency list per feature:

```json
{
  "milestones": [{
    "version": "v3.2",
    "name": "Idea-to-Shipped Journey",
    "waves": [{
      "id": "wave-1",
      "name": "Foundation",
      "features": [{
        "id": "feat-012",
        "slug": "seed-capture",
        "title": "Seed/Idea Capture",
        "status": "queued",
        "deps": [],
        "success_criteria": ["Can capture a braindump in under 2 minutes", "Seeds are searchable via RAG"]
      }]
    }]
  }]
}
```

**Dependency resolution:** Implement Kahn's algorithm inline in `lib/roadmap.js` (~40 lines). The wave graph has at most a few dozen nodes per milestone — no library needed. The function signature: `validateDeps(roadmap) → { valid: boolean, cycles: string[][] }`.

### Progress Tracking

Extend `lib/push-client.js` to sync wave completion to a new `wave_snapshots` Neon table. Extend `dashboard/lib/types.ts` with:

```typescript
interface WaveSnapshot {
  wave_id: string;
  milestone_version: string;
  project_name: string;
  name: string;
  feature_count: number;
  completed_count: number;
  completion_pct: number;
  velocity_7d: number; // features completed in last 7 days
  pushed_at: string;
}
```

### Discuss Phase State

**Location:** `.seraphim/discuss-state.json`

A simple lock file:

```json
{
  "locked": false,
  "decisions": [
    { "id": "DEC-001", "question": "...", "decision": "...", "locked_at": "2026-04-09T12:00:00Z" }
  ]
}
```

When `locked: true`, the planning phase checks this file before generating PLAN.md. The `/seraphim:discuss` command writes decisions here and flips the lock.

---

## Neon Schema Additions

Two new tables — DDL appended to `dashboard/scripts/migrate.ts`:

```sql
-- Research items (for cross-project research tracking)
CREATE TABLE IF NOT EXISTS research_items (
  id TEXT NOT NULL,
  project_name TEXT NOT NULL,
  question TEXT NOT NULL,
  category TEXT NOT NULL,
  status TEXT NOT NULL DEFAULT 'open',
  answer TEXT,
  source TEXT,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW(),
  PRIMARY KEY (id, project_name)
);

-- Wave progress snapshots (for dashboard progress visualization)
CREATE TABLE IF NOT EXISTS wave_snapshots (
  id SERIAL PRIMARY KEY,
  project_name TEXT NOT NULL,
  milestone_version TEXT NOT NULL,
  wave_id TEXT NOT NULL,
  name TEXT NOT NULL,
  feature_count INT NOT NULL DEFAULT 0,
  completed_count INT NOT NULL DEFAULT 0,
  velocity_7d FLOAT NOT NULL DEFAULT 0,
  pushed_at TIMESTAMPTZ DEFAULT NOW()
);
```

No new Neon SDK version needed. `@neondatabase/serverless` 1.0.2 is already installed and handles these tables.

---

## Dashboard New Pages

Two new Next.js pages — no new libraries:

| Page | Route | Chart Type | Data Source |
|------|-------|-----------|-------------|
| Roadmap Control Center | `/app/roadmap/page.tsx` | Nested list + horizontal bar per wave | `wave_snapshots` + `feature_snapshots` |
| Research Tracker | `/app/research/page.tsx` | Status badge list | `research_items` |

Chart.js horizontal bar chart is already available in 4.5.1. No new chart library needed. The roadmap tree view is a styled nested `<ul>` in Tailwind — no `react-flow` or graph layout library.

---

## What NOT to Add

| Avoid | Why | Instead |
|-------|-----|---------|
| `toposort` / `graphlib` | 30+ transitive deps for a 40-line algorithm; wave graphs have <50 nodes | Kahn's algorithm inline in `lib/roadmap.js` |
| `zod` | Not currently in project; runtime validation is light in this codebase | Inline type checks with clear error messages (existing pattern) |
| `react-flow` / `dagre` | 400KB+ bundle for a static roadmap tree; overkill | Nested HTML `<ul>` styled with Tailwind |
| `unified` / `remark` | Seeds and research docs are template-generated by AI agents; arbitrary markdown parsing not needed | Template-driven string generation (existing pattern in lib/) |
| Second vector store (Pinecone, Chroma) | RAG already in sqlite-vec; adding a second store creates sync burden | Re-use `lib/rag-indexer.js` — research docs indexed on write |
| `bull` / `p-queue` | Research is human-triggered; no background job queue needed | Synchronous command execution (existing pattern) |
| `@prisma/client` / Drizzle | Schema additions are two small tables; migration framework is overkill | `CREATE TABLE IF NOT EXISTS` in `migrate.ts` (v3.1 pattern) |

---

## Installation

No new packages. All capabilities already present.

```bash
# Verify existing deps are installed (should already be)
cd ~/.claude/plugins/seraphim && npm list better-sqlite3 @huggingface/transformers
cd ~/.claude/plugins/seraphim/dashboard && npm list chart.js @neondatabase/serverless next
```

---

## Sources

- Live codebase: `~/.claude/plugins/seraphim/package.json` — confirmed installed deps (HIGH)
- Live codebase: `~/.claude/plugins/seraphim/dashboard/package.json` — Chart.js 4.5.1, Next.js 16.2.3 (HIGH)
- Live codebase: `~/.claude/plugins/seraphim/lib/roadmap.js` — confirmed flat roadmap schema to extend (HIGH)
- Live codebase: `.planning/seeds/SEED-001-*.md` — confirmed seed file format already established (HIGH)
- Live codebase: `dashboard/lib/types.ts` — confirmed existing type structure for extension (HIGH)
- Kahn's algorithm: standard topological sort, well-documented, ~40 lines in Node.js (HIGH)

---
*Stack research for: Seraphim v3.2 Idea-to-Shipped Journey*
*Researched: 2026-04-09*
