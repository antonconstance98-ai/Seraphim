# Phase 32: Foundations - Context

**Gathered:** 2026-04-09
**Status:** Ready for planning

<domain>
## Phase Boundary

Clear all v3.1 technical debt (Neon DDL, schema consistency, feature_id wiring) and produce a schema extension audit confirming every v3.2 data concept maps to an existing structure. This is a prerequisite gate — no v3.2 feature work starts until this phase is verified.

</domain>

<decisions>
## Implementation Decisions

### Neon DDL Strategy
- **D-01:** Migration SQL files committed to repo at `dashboard/migrations/`. Each migration is idempotent (`CREATE TABLE IF NOT EXISTS`, `ALTER TABLE ... ADD COLUMN IF NOT EXISTS`).
- **D-02:** Migrations applied via a `migrate.ts` script that runs on deploy or manually via `npx tsx dashboard/migrations/migrate.ts`. No ORM — raw SQL through @neondatabase/serverless.
- **D-03:** PM tables needed: `projects`, `milestones`, `features`, `human_tasks`, `decisions`, `cost_snapshots`. These match the push-client.js payload structure. If tables already exist partially, alter to match.

### Schema Unification
- **D-04:** Standardize on `project_name` (snake_case string) as the canonical project identifier across ALL Neon tables and all push-client payloads. This matches the existing config convention.
- **D-05:** Any existing columns using `project` (without `_name`) get an `ALTER TABLE RENAME COLUMN` in the migration. Dashboard queries updated to match.
- **D-06:** In-code JavaScript uses camelCase `projectName` per JS conventions. Snake_case `project_name` only in SQL and JSON payloads.

### feature_id Wiring
- **D-07:** Trace all callers of `decisions-logger.js` buildRecord(). Any caller in the pipeline that has access to the current feature context must pass `feature_id`. Callers without feature context continue passing `null` (existing default).
- **D-08:** The feature_id comes from `.seraphim/roadmap.json` active feature. The `pm-context.js` module already exposes this — wire it through to decisions-logger at the call sites.

### Schema Extension Audit
- **D-09:** Audit document written to `.planning/phases/32-foundations/SCHEMA-AUDIT.md`. Maps each v3.2 data concept to its home in existing structures:
  - Seeds → `.planning/seeds/` (exists) + index.jsonl (new file)
  - Requirements → `.seraphim/requirements.json` (new file, follows roadmap.json pattern)
  - Waves → extend `roadmap.json` features with `waves[]` array
  - Discuss decisions → `.planning/phases/NN-slug/NN-CONTEXT.md` (exists)
  - Research items → `.seraphim/research.json` (new file)
  - Human task enrichment → extend existing HumanTask type with optional fields
  - Progress/velocity → computed from existing timeline.jsonl + task-completions.jsonl
- **D-10:** The audit confirms "extend, not duplicate" for every concept. If a concept would require a new top-level data store, it must justify why the existing store can't be extended.

### Claude's Discretion
- All four gray areas (Neon DDL strategy, schema unification, extension audit format, feature_id wiring) — user gave full discretion.

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Plugin Lib Files
- `~/.claude/plugins/seraphim/lib/roadmap.js` — Atomic write pattern, readRoadmap/writeRoadmap, nextFeatureId — canonical reference for new lib files
- `~/.claude/plugins/seraphim/lib/decisions-logger.js` — buildRecord() with feature_id parameter (line 19) — wiring target
- `~/.claude/plugins/seraphim/lib/push-client.js` �� Neon push payload structure, project_name usage (lines 78, 261)
- `~/.claude/plugins/seraphim/lib/pm-context.js` — PM context including active feature — source of feature_id

### Dashboard
- `dashboard/` — Next.js app that queries Neon; any DDL changes here
- `dashboard/lib/types.ts` — TypeScript types for Neon schema (if exists)

### Existing Data Files
- `.seraphim/roadmap.json` — Current milestone-feature hierarchy to extend
- `.seraphim/config.json` — Project config with project_name

### Research
- `.planning/research/SUMMARY.md` — v3.2 research summary with pitfall warnings
- `.planning/research/PITFALLS.md` — Detailed pitfall analysis (Pitfall 1: duplicate primitives, Pitfall 3: Neon divergence)

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `roadmap.js` atomic write pattern (write to tmp, rename) — use for all new .seraphim/ files
- `push-client.js` fire-and-forget Neon push — extend payload, don't create new push mechanism
- `pm-context.js` feature context loading — source of feature_id for wiring

### Established Patterns
- Snake_case in SQL/JSON payloads, camelCase in JavaScript
- `CREATE TABLE IF NOT EXISTS` for idempotent DDL
- `readX/writeX` function pairs for JSON file CRUD (roadmap.js pattern)

### Integration Points
- decisions-logger.js call sites need feature_id injection
- push-client.js payload needs standardized project_name
- Dashboard queries need column name updates after rename

</code_context>

<specifics>
## Specific Ideas

No specific requirements — user gave full Claude discretion on all infrastructure decisions.

</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope

</deferred>

---

*Phase: 32-foundations*
*Context gathered: 2026-04-09*
