# Phase 36: Human Tasks + Debugging - Research

**Researched:** 2026-04-10
**Domain:** Seraphim plugin command authoring, Neon DDL migration, debug-state persistence
**Confidence:** HIGH

## Summary

Phase 36 splits into two independent tracks: (1) enriching the existing `human_tasks` schema and all callers with three optional text fields (`skills_to_learn`, `thought_prompt`, `research_task`), and (2) building two new command files (`debug.md`, `forensics.md`) backed by a lib module for debug-state persistence in `.planning/debug/`.

The enrichment track is pure additive — one SQL migration adds three nullable TEXT columns, `push-client.js` scanPendingTasks adds three optional attributes to the SERAPHIM:HUMAN_TASKS marker regex reader, the ingest route.ts INSERT extends its column list, and the inbox.md display loop adds inline rendering for the three fields. Pipeline injection happens at marker-emission time in `run.md`'s Step 6e gate logic.

The debug/forensics track introduces two new commands. `debug.md` reads and appends to a per-slug `.md` file with YAML frontmatter in `.planning/debug/`. `forensics.md` is a read-only subagent with restricted tools that writes a report to `.planning/debug/forensics/`. Auto-repair logic (DBG-04) should live in a `lib/repair.js` strategy cascade, not embedded in a command file.

**Primary recommendation:** Implement both tracks as independent waves. Wave 1 completes the schema migration and enrichment plumbing. Wave 2 implements debug.md, forensics.md, and repair.js. There are zero npm dependencies needed — all patterns already exist in the codebase.

---

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions
- **D-01:** Extend existing HumanTask type in `push-client.js` payload with optional `skills_to_learn` (string[]), `thought_prompt` (string), `research_task` (string) fields. Neon `human_tasks` table gets matching TEXT columns via new migration.
- **D-02:** Enrichment fields populated at task creation time — when pipeline phases (Judge, Crucible) create human tasks, they analyze context and populate if relevant. Null if not applicable.
- **D-03:** Extend existing `/seraphim:inbox` command to display enrichment fields inline. Skills as tags, thought-prompt as expandable section.
- **D-04:** Debug state persists in `.planning/debug/{slug}.md` with YAML frontmatter (status, hypothesis, findings, timeline). Each `/seraphim:debug` session reads existing state and appends new findings.
- **D-05:** Auto-repair uses strategy cascade — RETRY (re-run failed task), DECOMPOSE (split into subtasks), PRUNE (remove blocked dependency), ESCALATE (surface to human). Budget: max 2 retries, 1 decompose before escalation.
- **D-06:** `/seraphim:forensics` produces read-only root-cause report. Analyzes git history, error logs, state files. Writes to `.planning/debug/forensics/`. No code changes, no commits. Subagent with restricted tool access.
- **D-07:** Debug sessions can be spawned from UAT gap items — when verification finds gaps, it creates debug sessions linked to specific verification failures.

### Claude's Discretion
- Debug report formatting, auto-repair budget thresholds, and enrichment heuristics at Claude's discretion.

### Deferred Ideas (OUT OF SCOPE)
- None — discussion stayed within phase scope
</user_constraints>

---

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| HTASK-01 | Human task inbox enriched with skills-to-learn field | SQL migration + marker attr + push-client + ingest route + inbox display |
| HTASK-02 | Human task inbox enriched with thought-prompt field | Same pipeline as HTASK-01 |
| HTASK-03 | Human task inbox enriched with research-task field | Same pipeline as HTASK-01 |
| DBG-01 | User can debug systematically via `/seraphim:debug` with persistent state across resets | `.planning/debug/{slug}.md` YAML frontmatter pattern confirmed in codebase |
| DBG-02 | Autonomous root-cause analysis agents for UAT gaps | Subagent dispatch pattern established in Phase 33 (autonomous.md) |
| DBG-03 | User can run post-mortem investigation via `/seraphim:forensics` (read-only, diagnostic) | Restricted subagent pattern, report writes to `.planning/debug/forensics/` |
| DBG-04 | Failed task auto-repair with RETRY/DECOMPOSE/PRUNE/ESCALATE strategies | lib/repair.js strategy cascade; no npm deps needed |
</phase_requirements>

---

## Standard Stack

### Core
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| Node.js | v22.22.0 (installed) | All lib scripts, command inline scripts | Matches all existing hooks and lib files |
| `@neondatabase/serverless` | Current (dashboard) | Neon DDL migration + ingest route | Already used in dashboard/app/api/ingest/route.ts |
| fs / path / os (Node stdlib) | Built-in | File I/O for debug state, push-client | Zero-dependency pattern is a project invariant |

### Supporting
| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| YAML frontmatter (manual parse) | — | Debug state in `.planning/debug/*.md` | Existing pattern: `parseFrontmatter` in lib/; do NOT add a yaml npm package |

### Alternatives Considered
| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| Manual YAML frontmatter parse | `js-yaml` npm package | Project invariant: zero new npm dependencies — always hand-parse |
| Inline SQL in route.ts | ORM | Existing pattern uses raw tagged-template sql`` — stay consistent |

**Installation:** No new packages. Zero npm changes required.

---

## Architecture Patterns

### Human Task Enrichment — Data Flow

```
run.md (Step 6e gate)
  └─ emitMarker('HUMAN_TASKS', { ..., skills_to_learn, thought_prompt, research_task })
       └─ forge-log.md (appended)
            └─ push-client.js scanPendingTasks() reads marker attrs
                 └─ payload.human_tasks[] with 3 new optional fields
                      └─ /api/ingest route.ts INSERT adds 3 columns
                           └─ Neon human_tasks table (migration 002)
```

`inbox.md` reads `forge-log.md` directly (not via Neon) — it also needs the three fields parsed and displayed.

### Debug State File Pattern

Existing example confirmed at `.planning/debug/seraphim-plugin-not-loading.md`:

```yaml
---
status: awaiting_human_verify
trigger: "..."
created: ISO8601
updated: ISO8601
---

## Current Focus
hypothesis: ...
test: ...
expecting: ...
next_action: ...

## Symptoms / Evidence / Eliminated / Resolution
```

`/seraphim:debug` reads existing slug file (or creates it), appends a new `## Session {N}` block with new findings, updates `updated` frontmatter timestamp, and rewrites the file atomically.

### Pattern 1: SERAPHIM Marker Attribute Extension
**What:** Add optional attrs to existing `emitMarker('HUMAN_TASKS', ...)` call in `run.md` Step 6e
**When to use:** Adding data to existing marker types — never create a parallel marker type
**Example:**
```javascript
// Source: run.md Step 6e existing pattern
const marker = markers.emitMarker('HUMAN_TASKS', {
  id: `T-${PHASE_ID}-${GATE_NAME}`,
  assignee: 'human',
  type: TASK_TYPE,
  status: 'pending',
  description: DESCRIPTION,
  // NEW — all optional, omit if null:
  skills_to_learn: skills ? skills.join(',') : undefined,
  thought_prompt: thoughtPrompt || undefined,
  research_task: researchTask || undefined
});
```

Parser in `scanPendingTasks` reads these back with the existing `get(key)` regex pattern.

### Pattern 2: Neon Migration File
**What:** New SQL file `002-human-tasks-enrichment.sql` — additive ALTER TABLE only
**When to use:** Any schema change
**Example:**
```sql
-- 002-human-tasks-enrichment.sql
ALTER TABLE human_tasks ADD COLUMN IF NOT EXISTS skills_to_learn TEXT;
ALTER TABLE human_tasks ADD COLUMN IF NOT EXISTS thought_prompt   TEXT;
ALTER TABLE human_tasks ADD COLUMN IF NOT EXISTS research_task    TEXT;
```
`IF NOT EXISTS` guard makes it idempotent — safe to run twice.

### Pattern 3: Ingest Route Extension
**What:** Extend the `INSERT INTO human_tasks` in `route.ts` with three new optional columns
**When to use:** Schema column additions
**Example:**
```typescript
// Source: dashboard/app/api/ingest/route.ts existing pattern
await sql`
  INSERT INTO human_tasks
    (project_name, task_id, type, status, feature_id, urgency,
     skills_to_learn, thought_prompt, research_task)
  VALUES
    (${payload.project_name}, ${t.task_id}, ${t.type}, ${t.status},
     ${t.feature_id ?? null}, ${t.urgency},
     ${t.skills_to_learn ?? null}, ${t.thought_prompt ?? null}, ${t.research_task ?? null})
  ON CONFLICT (project_name, task_id) DO UPDATE
  SET type = EXCLUDED.type, status = EXCLUDED.status,
      feature_id = EXCLUDED.feature_id, urgency = EXCLUDED.urgency,
      skills_to_learn = EXCLUDED.skills_to_learn,
      thought_prompt = EXCLUDED.thought_prompt,
      research_task = EXCLUDED.research_task,
      synced_at = NOW()
`;
```

### Pattern 4: lib/repair.js Strategy Cascade
**What:** Pure Node.js module exporting `attemptRepair(task, context)` with strategy cascade
**When to use:** DBG-04 — auto-repair of failed tasks
**Example (structure):**
```javascript
// lib/repair.js — no file I/O side effects; caller writes state
const STRATEGIES = ['RETRY', 'DECOMPOSE', 'PRUNE', 'ESCALATE'];

function selectStrategy(task, repairHistory) {
  const retries = repairHistory.filter(h => h.strategy === 'RETRY').length;
  const decomposes = repairHistory.filter(h => h.strategy === 'DECOMPOSE').length;
  if (retries < 2) return 'RETRY';
  if (decomposes < 1) return 'DECOMPOSE';
  return 'ESCALATE';
}
module.exports = { selectStrategy, STRATEGIES };
```

Budget thresholds (D-05): max 2 retries, 1 decompose, then ESCALATE.

### Anti-Patterns to Avoid
- **Embedding repair logic in debug.md:** Business logic belongs in `lib/repair.js`. Commands are thin orchestrators.
- **Creating a new marker type for enriched tasks:** Extend existing `HUMAN_TASKS` marker attributes — do not create `HUMAN_TASKS_ENRICHED`.
- **Writing YAML with a parser library:** Use string templates for frontmatter writes; use simple regex/split for reads. No `js-yaml` dependency.
- **Making forensics.md write code changes:** D-06 is strict — forensics is read-only. No Write to source files, no Bash mutations, no commits.

---

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Debug file atomic write | Custom rename dance | Follow `lib/roadmap.js` tmp+rename pattern already in codebase | Race-condition-safe; established pattern |
| Marker attribute parsing | New regex from scratch | Existing `get(key)` closure pattern in `scanPendingTasks` in `push-client.js` | Already tested and deployed |
| Neon migration running | Custom migration runner | Update `dashboard/migrations/migrate.ts` to include new file | Already handles idempotent migrations |

**Key insight:** Every piece of infrastructure already exists. This phase is purely additive — new columns, new attrs, new command files that follow established patterns.

---

## Common Pitfalls

### Pitfall 1: Marker attrs with array values
**What goes wrong:** `skills_to_learn` is `string[]` in the type but HTML comment attrs are strings. Arrays stored in a single attr become unparseable.
**Why it happens:** Marker format is `key="val"` only — no JSON, no arrays.
**How to avoid:** Serialize as comma-separated: `skills_to_learn="TypeScript,SQL"`. Deserialize with `.split(',').map(s => s.trim())` on the read side.
**Warning signs:** Inbox display shows raw comma string instead of tags.

### Pitfall 2: NULL vs undefined in ingest route
**What goes wrong:** TypeScript types don't match Neon nullable columns — `undefined` is not `null` and causes runtime errors.
**Why it happens:** JS `undefined` doesn't serialize to SQL NULL.
**How to avoid:** Always use `t.field ?? null` in the tagged-template interpolation — never `t.field || null` (which coerces empty string to null unexpectedly).

### Pitfall 3: Debug file collision on concurrent sessions
**What goes wrong:** Two `/seraphim:debug` calls on same slug overwrite each other.
**Why it happens:** Read-modify-write is not atomic without tmp+rename.
**How to avoid:** Use the tmp+rename atomic write pattern: write to `{slug}.tmp.md` then `fs.renameSync` to `{slug}.md`.

### Pitfall 4: Forensics subagent with unrestricted tools
**What goes wrong:** Agent writes to source files, breaking the read-only contract.
**Why it happens:** Default allowed-tools is too permissive.
**How to avoid:** `forensics.md` frontmatter must set `allowed-tools: ["Read", "Bash"]` where Bash is restricted to read-only commands (git log, grep, cat). Explicitly state in instructions: no Write, no Edit, no commits.

### Pitfall 5: Enrichment heuristic at display time vs. creation time
**What goes wrong:** Trying to populate `thought_prompt` by reading the task at inbox display time — this requires AI context that isn't available at that point.
**Why it happens:** D-02 says populate at task creation time. Display is purely reading stored data.
**How to avoid:** Enrichment logic lives in `run.md` Step 6e (at marker emission time). `inbox.md` only renders whatever is already stored in the marker.

### Pitfall 6: Migration not applied to production Neon
**What goes wrong:** `route.ts` inserts new columns but Neon schema doesn't have them — runtime 500 errors.
**Why it happens:** `002-human-tasks-enrichment.sql` must be manually applied (same as `001`).
**How to avoid:** Add a human task in the plan reminding the user to apply the migration via `migrate.ts` before testing push.

---

## Code Examples

### inbox.md enrichment display (inline rendering)
```javascript
// After printing task ID and description, check for enrichment fields
const skillsAttr = t.skills_to_learn;
const thoughtAttr = t.thought_prompt;
const researchAttr = t.research_task;

if (skillsAttr) {
  const tags = skillsAttr.split(',').map(s => s.trim()).filter(Boolean);
  console.log('       Skills: ' + tags.map(s => '[' + s + ']').join(' '));
}
if (thoughtAttr) {
  console.log('       Think:  ' + thoughtAttr);
}
if (researchAttr) {
  console.log('       Research: ' + researchAttr);
}
```

### debug.md — read-then-append pattern
```javascript
const debugDir = path.join(planningRoot, 'debug');
const slugFile = path.join(debugDir, slug + '.md');
let existingContent = '';
if (fs.existsSync(slugFile)) {
  existingContent = fs.readFileSync(slugFile, 'utf8');
}
// Parse frontmatter, update 'updated' field, append new session block
// Write via tmp+rename for atomicity
const tmpFile = slugFile + '.tmp';
fs.writeFileSync(tmpFile, updatedContent, 'utf8');
fs.renameSync(tmpFile, slugFile);
```

---

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Bare human_tasks columns (6) | + 3 enrichment columns | Phase 36 | Inbox becomes a coaching tool |
| No debug persistence | `.planning/debug/{slug}.md` YAML state | Phase 36 | Debug survives session resets |
| No auto-repair | RETRY/DECOMPOSE/PRUNE/ESCALATE cascade | Phase 36 | Failed tasks self-heal within budget |

---

## Environment Availability

Step 2.6: No new external dependencies identified. All tooling (Node.js, Neon driver, git) already confirmed installed in prior phases.

The Neon migration (`002-human-tasks-enrichment.sql`) requires production Neon access — same manual step as Phase 32. This is not a blocker for development/testing of the command logic.

---

## Validation Architecture

Test infrastructure: No automated test framework detected in this plugin (all prior phases used manual verification). This matches the established pattern — validation is done by running commands and observing output.

### Phase Requirements → Test Map

| Req ID | Behavior | Test Type | Validation Command |
|--------|----------|-----------|-------------------|
| HTASK-01 | skills_to_learn appears in inbox output | Manual smoke | Run `/seraphim:inbox` after emitting a marker with skills attr |
| HTASK-02 | thought_prompt appears in inbox output | Manual smoke | Same as above with thought_prompt attr |
| HTASK-03 | research_task appears in inbox output | Manual smoke | Same as above with research_task attr |
| DBG-01 | `/seraphim:debug` creates/appends `.planning/debug/{slug}.md` | Manual smoke | Run command twice, verify file grows and frontmatter `updated` changes |
| DBG-02 | Root-cause agent spawned from UAT gap | Manual smoke | Trigger a UAT gap scenario and confirm debug file created |
| DBG-03 | `/seraphim:forensics` writes to `.planning/debug/forensics/` | Manual smoke | Run command, confirm report file present, confirm no source files modified |
| DBG-04 | repair.js selectStrategy respects budget (2 retries max) | Unit test (inline node -e) | `node -e` script calling selectStrategy with mocked repair history |

### Wave 0 Gaps

None for test framework (no framework needed). The `repair.js` strategy logic can be validated with a quick inline node script in the plan.

---

## Sources

### Primary (HIGH confidence)
- Direct code inspection: `/home/alucardmessangeroflight/.claude/plugins/seraphim/lib/push-client.js` — confirmed scanPendingTasks shape, marker regex, payload structure
- Direct code inspection: `/home/alucardmessangeroflight/.claude/plugins/seraphim/dashboard/app/api/ingest/route.ts` — confirmed INSERT column list and ON CONFLICT clause
- Direct code inspection: `/home/alucardmessangeroflight/.claude/plugins/seraphim/dashboard/migrations/001-initial-pm-schema.sql` — confirmed `human_tasks` schema (6 columns)
- Direct code inspection: `/home/alucardmessangeroflight/.claude/plugins/seraphim/commands/inbox.md` — confirmed display loop structure
- Direct code inspection: `/home/alucardmessangeroflight/.claude/plugins/seraphim/commands/run.md` — confirmed Step 6e marker emission pattern
- Direct code inspection: `/home/alucardmessangeroflight/.claude/plugins/seraphim/lib/markers.js` — confirmed emitMarker API
- Direct code inspection: `/home/alucardmessangeroflight/projects/seraphim/.planning/debug/seraphim-plugin-not-loading.md` — confirmed debug YAML frontmatter schema in use

### Secondary (MEDIUM confidence)
- CONTEXT.md decisions D-01 through D-07 — authoritative for all locked choices

---

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH — all libraries already in production use; no new deps
- Architecture: HIGH — patterns confirmed by reading actual deployed code
- Pitfalls: HIGH — pitfalls derived from inspecting actual code paths (marker attr format, NULL handling, atomic write gap)

**Research date:** 2026-04-10
**Valid until:** 2026-05-10 (stable — no fast-moving dependencies)
