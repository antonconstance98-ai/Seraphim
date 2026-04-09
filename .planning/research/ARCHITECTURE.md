# Architecture Research

**Domain:** Idea-to-shipped workflow layer on top of an existing AI multi-model plugin
**Researched:** 2026-04-09
**Confidence:** HIGH (all conclusions drawn from direct inspection of live codebase)

## Standard Architecture

### System Overview

```
┌──────────────────────────────────────────────────────────────────────┐
│                     Human Entry Points (new)                         │
│  /seraphim:seed  /seraphim:research  /seraphim:requirements          │
│  /seraphim:discuss  /seraphim:plan   /seraphim:verify                │
└────────────────────────────┬─────────────────────────────────────────┘
                             │  slash commands (markdown, same pattern)
┌────────────────────────────▼─────────────────────────────────────────┐
│                  Existing Command Layer (extend only)                 │
│  new-project  new-milestone  add-feature  start  roadmap  tasks      │
│  discover  envision  judge  architect  forge  crucible               │
└──────────┬──────────────────────────────────┬────────────────────────┘
           │                                  │
┌──────────▼──────────┐             ┌─────────▼─────────────────────┐
│  NEW: Idea Store    │             │  EXISTING: Phase State Store   │
│  .planning/seeds/   │             │  .seraphim/phases/{id}/        │
│  .planning/research/│             │  state.json, blueprint.md      │
│  .seraphim/reqs.json│             │  forge-log.md                  │
│  .seraphim/discuss/ │             └─────────────────────────────────┘
└──────────┬──────────┘
           │
┌──────────▼──────────────────────────────────────────────────────────┐
│                  EXISTING: Core Lib Layer (extend only)              │
│  roadmap.js   decisions-logger.js   phase-state.js                  │
│  push-client.js   progress-extractor.js   markers.js                │
│  NEW: requirements.js   research-tracker.js   wave-planner.js       │
└──────────┬──────────────────────────────────────────────────────────┘
           │
┌──────────▼──────────────────────────────────────────────────────────┐
│            EXISTING: Persistence Layer (two new tables only)        │
│  .seraphim/roadmap.json   decisions.jsonl   token-log.jsonl         │
│  NEW: .seraphim/requirements.json   .seraphim/waves.json            │
│  Neon: NEW requirement_snapshots + wave_snapshots tables            │
└──────────┬──────────────────────────────────────────────────────────┘
           │  push-client.js (existing fire-and-forget pattern)
┌──────────▼──────────────────────────────────────────────────────────┐
│              EXISTING: Dashboard (Next.js, extend only)             │
│  NEW panels: requirements tree, wave progress bars, velocity chart  │
│  NEW pages: /projects/[slug]/roadmap  /projects/[slug]/plan         │
└─────────────────────────────────────────────────────────────────────┘
```

### Component Responsibilities

| Component | Responsibility | Status |
|-----------|----------------|--------|
| `.planning/seeds/` | Dormant ideas in SEED-NNN.md format with frontmatter | EXISTS — extend format |
| `.planning/research/` | Research files written by researcher agents | EXISTS — already used |
| `.seraphim/requirements.json` | REQ-ID scoped requirements (v1/future/oos) | NEW file schema |
| `.seraphim/discuss/` | Lock-and-sign decision records before planning | NEW directory |
| `.seraphim/waves.json` | Wave-structured plan: wave N with tasks, deps, done-criteria | NEW file schema |
| `lib/requirements.js` | CRUD for requirements.json, REQ-ID generation | NEW lib |
| `lib/wave-planner.js` | Read/write waves.json, dependency resolution | NEW lib |
| `lib/research-tracker.js` | Track research items, interrogation state, completion | NEW lib |
| `commands/seed.md` | Braindump capture, merge into seeds dir | NEW command |
| `commands/research.md` | Human interrogation then AI research orchestration | NEW command |
| `commands/requirements.md` | Define/scope requirements, assign REQ-IDs | NEW command |
| `commands/discuss.md` | Decision lockdown before planning begins | NEW command |
| `commands/plan.md` | Generate wave-structured PLAN.md from requirements + roadmap | NEW command |
| `commands/verify.md` | Goal-backward verification, UAT checklist | NEW command |
| `push-client.js` | Sync all data to Neon (extend with new tables) | EXISTS — extend |
| `progress-extractor.js` | Supply velocity/completion data (extend for waves) | EXISTS — extend |
| Dashboard panels | Progress bars, velocity, roadmap tree, wave view | EXISTS — extend |

## Recommended Project Structure (delta only)

```
~/.claude/plugins/seraphim/
├── commands/
│   ├── seed.md              # NEW — braindump capture
│   ├── research.md          # NEW — human interrogation + AI research
│   ├── requirements.md      # NEW — REQ-ID scoping
│   ├── discuss.md           # NEW — decision lockdown
│   ├── plan.md              # NEW — wave-structured PLAN.md generation
│   ├── verify.md            # NEW — goal-backward verification + UAT
│   └── [existing commands unchanged]
├── lib/
│   ├── requirements.js      # NEW — requirements.json CRUD, REQ-ID generation
│   ├── wave-planner.js      # NEW — waves.json CRUD, dep resolution
│   ├── research-tracker.js  # NEW — research item state tracking
│   └── [existing libs unchanged]
└── skills/
    └── [existing skills unchanged]

per-project .seraphim/
├── requirements.json        # NEW — REQ-ID indexed requirements with scope
├── waves.json               # NEW — wave-structured plan
├── discuss/                 # NEW — decision records (one .md per decision)
│   └── DECISION-NNN.md
└── [existing files unchanged]

per-project .planning/
├── seeds/                   # EXISTS — extend SEED frontmatter with v3.2 fields
├── research/                # EXISTS — already populated by researcher agents
└── [existing files unchanged]
```

### Structure Rationale

- **New lib files only:** All existing libs (roadmap.js, phase-state.js, decisions-logger.js) are untouched. New features get new files. This prevents regression in the six-phase pipeline.
- **New commands only:** Existing commands (discover, forge, etc.) are untouched. New commands follow the identical markdown-instruction pattern.
- **.seraphim/ for runtime data:** requirements.json and waves.json live in .seraphim/ (per-project runtime state) not .planning/ (human workspace). This matches how roadmap.json is handled.
- **.planning/seeds/ already exists:** The SEED-NNN.md format is proven. The seed command writes to it without structural change.

## Architectural Patterns

### Pattern 1: Markdown Command with Node.js Lib Call

The universal pattern for every existing Seraphim command. New commands must follow it exactly.

**What:** A command.md file contains natural-language instructions for Claude. When resolution of state is needed it shells out to a Node.js lib file via inline bash.
**When to use:** Every new slash command — seed, research, requirements, discuss, plan, verify.
**Trade-offs:** Claude reads the instructions and acts as the executor, which keeps complex logic readable. The lib call is only for file I/O that needs to be deterministic.

**Example (cloned from new-milestone.md):**
```bash
node -e "
  const r = require('${PLUGIN_ROOT}/lib/requirements.js');
  const reqs = r.readRequirements('${PROJECT_ROOT}');
  console.log(JSON.stringify(reqs));
"
```

### Pattern 2: Append-Only JSONL for Events, JSON for Current State

**What:** Mutable current state lives in a .json file (roadmap.json, waves.json, requirements.json). Events and audit trail go to .jsonl (decisions.jsonl, token-log.jsonl).
**When to use:** requirements.json holds current scoped requirements. waves.json holds current plan. A change-log.jsonl can hold history if needed, but is not required for MVP.
**Trade-offs:** JSON is easy to read and write atomically (temp-file rename pattern established in roadmap.js). JSONL is append-only and never corrupts on partial write.

### Pattern 3: Fire-and-Forget Dashboard Push

**What:** push-client.js collects project state and POSTs to SERAPHIM_DASHBOARD_URL. Never throws. New data types (requirements, waves) are added as additional fields in the same POST body.
**When to use:** After any command that mutates requirements.json or waves.json.
**Trade-offs:** Dashboard data can be stale by one pipeline step, which is acceptable. No synchronous dependency on dashboard availability.

### Pattern 4: REQ-ID and WAVE-ID as Stable References

**What:** requirements.json assigns REQ-001, REQ-002 IDs. waves.json assigns WAVE-1, WAVE-2 with tasks referencing REQ-IDs. This mirrors feat-NNN in roadmap.js.
**When to use:** All cross-file references. REQ-IDs appear in plan tasks, discuss decisions, and verify checklists.
**Trade-offs:** Sequential IDs are simple to generate (clone nextFeatureId pattern from roadmap.js). They survive renames because the ID never changes.

## Data Flow

### Idea-to-Shipped Flow

```
/seraphim:seed
    writes .planning/seeds/SEED-NNN.md
    |
/seraphim:research [topic]
    human interrogation -> AI research agents -> .planning/research/
    |
/seraphim:requirements
    reads research + seeds -> writes .seraphim/requirements.json (REQ-IDs)
    |
/seraphim:discuss
    reads requirements -> writes .seraphim/discuss/DECISION-NNN.md (locked)
    |
/seraphim:plan
    reads requirements + discuss + roadmap -> writes .seraphim/waves.json + PLAN.md
    |
/seraphim:start {feature}  [EXISTING]
    reads waves.json for task context -> enters six-phase pipeline
    |
/seraphim:verify
    reads waves.json done-criteria + phase outputs -> UAT checklist + verdict
```

### Dashboard Push Flow

```
Any mutating command completes
    |
push-client.js called (existing pattern)
    serializes: config + roadmap + requirements + waves + decisions
    POSTs to SERAPHIM_DASHBOARD_URL (fire-and-forget)
    |
Neon ingest endpoint writes to:
    existing: feature_snapshots, human_task_snapshots
    new:      requirement_snapshots, wave_snapshots
    |
Dashboard Next.js Server Components query Neon
    renders: wave progress bars, completion %, velocity, roadmap tree
```

### Progress Visualization Flow

```
progress-extractor.js (EXTENDED)
    reads: phase state.json (existing) + waves.json (new)
    returns: {
      phases_completed, phases_total,       // existing fields
      waves_completed, waves_total,          // new fields
      tasks_done, tasks_total,               // extended from retries count
      velocity_tasks_per_day,               // new — derived from timestamps
      current_wave                           // new
    }
```

## Integration Points

### New vs Modified

| Component | New or Modified | Integration Note |
|-----------|----------------|-----------------|
| `commands/seed.md` | NEW | Writes to .planning/seeds/ — no existing command conflicts |
| `commands/research.md` | NEW | Orchestrates researcher subagent using seraphim-discover.md agent pattern |
| `commands/requirements.md` | NEW | Reads seeds + research, writes .seraphim/requirements.json |
| `commands/discuss.md` | NEW | Reads requirements.json, writes .seraphim/discuss/ directory |
| `commands/plan.md` | NEW | Reads requirements + discuss + roadmap.json, writes waves.json |
| `commands/verify.md` | NEW | Reads waves.json done-criteria, runs goal-backward check |
| `lib/requirements.js` | NEW | CRUD for requirements.json, REQ-ID generation (clone nextFeatureId pattern) |
| `lib/wave-planner.js` | NEW | CRUD for waves.json, dependency graph resolution |
| `lib/research-tracker.js` | NEW | Track open/closed research items in .planning/research/ |
| `lib/push-client.js` | MODIFIED | Add requirements + waves to the serialized payload |
| `lib/progress-extractor.js` | MODIFIED | Add wave-level progress fields to return object |
| `commands/tasks.md` | MODIFIED | Show wave tasks alongside phase blueprint tasks |
| `dashboard/app/` | MODIFIED | New panels and pages for requirements, waves, velocity |
| `plugin.json` | NO CHANGE | Commands dir auto-discovered; no manifest update needed |
| `.seraphim/config.json` | NO CHANGE | No new config fields required for MVP |
| Six-phase commands | NO CHANGE | discover, envision, judge, architect, forge, crucible untouched |
| Executor layer | NO CHANGE | dispatch.js and all model executors untouched |

### Internal Boundaries

| Boundary | Communication | Notes |
|----------|---------------|-------|
| requirements.md ↔ requirements.js | Bash node -e inline call | Same pattern as all existing commands |
| plan.md ↔ wave-planner.js | Bash node -e inline call | Also reads roadmap.json via existing roadmap.js |
| plan.md ↔ discuss/ | Read tool on discuss/DECISION-NNN.md | Decisions are locked MD files, Claude reads them directly |
| push-client.js ↔ Neon | HTTP POST (existing pattern) | Add new table rows to same payload object |
| verify.md ↔ waves.json | wave-planner.js read call | done-criteria per task become the verification checklist |
| research.md ↔ researcher agent | Spawn subagent (existing pattern from seraphim-discover.md) | Agent writes .planning/research/ files |

### Enriched Human Tasks Integration

The existing markers.js system handles SERAPHIM:HUMAN_TASK markers in blueprint.md. For enriched tasks, add a `task_type` field to the marker metadata — no changes to markers.js core required:

- `research` — "go learn X before continuing"
- `skill` — "this requires learning Y; resources provided"
- `decision` — "human must decide Z" (feeds discuss command)
- `validation` — "UAT step: verify W manually"

The `task_type` convention is enforced by command instructions, not by lib code. markers.js already preserves unknown fields in the metadata object.

## Scaling Considerations

| Scale | Architecture Adjustments |
|-------|--------------------------|
| Solo developer (current) | Current approach is correct — JSONL/JSON files, single Neon DB, no queues needed |
| Team of 3-5 | Add file locking for requirements.json writes; optimistic concurrency on waves.json |
| Multi-team | Per-project Neon schemas; user attribution fields in REQ-IDs and task completions |

This project targets a solo developer. Multi-user concerns are explicitly out of scope.

## Anti-Patterns

### Anti-Pattern 1: Blocking the Six-Phase Pipeline on PM State

**What people do:** Add a requirements check gate inside discover.md or architect.md — "must have REQ-IDs before proceeding."
**Why it's wrong:** The six-phase pipeline must remain self-contained. Injecting PM validation breaks the existing workflow for projects that bypass the new commands.
**Do this instead:** The plan.md command prepares context. The human invokes /seraphim:start only after plan is done. The pipeline itself is never gated.

### Anti-Pattern 2: Embedding Wave Logic Inside Command Markdown

**What people do:** Put dependency resolution and REQ-ID generation logic inside command .md as inline JavaScript in bash heredocs.
**Why it's wrong:** Markdown instructions are for Claude to read. Complex logic in heredoc bash strings is hard to test, hard to debug, and breaks on quote escaping.
**Do this instead:** All logic goes in lib/wave-planner.js and lib/requirements.js. Commands call the lib with a single node -e line.

### Anti-Pattern 3: New Database for the Requirements Layer

**What people do:** Add SQLite or a second Neon schema for requirements, research items, and wave plans.
**Why it's wrong:** The existing JSONL/JSON + Neon pattern works at this scale. Adding a second persistence layer creates two sources of truth and doubles the sync burden.
**Do this instead:** requirements.json and waves.json in .seraphim/ as source of truth. Neon receives snapshots via the existing push-client pattern.

### Anti-Pattern 4: Modifying New-Project for PM Scaffolding

**What people do:** Modify new-project.md to scaffold requirements.json and waves.json during project creation.
**Why it's wrong:** New-project is the bootstrap command used before any milestone context exists. Mixing PM scaffolding into it creates ordering confusion.
**Do this instead:** requirements.json and waves.json are created on first use of requirements.md and plan.md respectively. Lazy initialization matches all other .seraphim/ files.

## Build Order

The dependency graph dictates this order. Each item depends on the ones above it.

**Wave 1 — Data foundations (no UI, no research)**
1. `lib/requirements.js` — REQ-ID CRUD (no deps, clone roadmap.js pattern)
2. `lib/wave-planner.js` — waves.json CRUD with dep resolution (depends on requirements.js for REQ-ID validation)
3. `commands/seed.md` — writes to existing .planning/seeds/ (no new lib needed)
4. `commands/requirements.md` — uses requirements.js (depends on item 1)
5. `commands/discuss.md` — reads requirements.json, writes discuss/ dir (depends on item 4)
6. `commands/plan.md` — uses wave-planner.js + roadmap.js (depends on items 2, 5)

**Wave 2 — Research, verification, enriched tasks**
7. `lib/research-tracker.js` — tracks research item state in .planning/research/
8. `commands/research.md` — orchestrates researcher agent (depends on item 7)
9. `commands/verify.md` — reads waves.json done-criteria (depends on item 2)
10. Extend `commands/tasks.md` task_type convention for enriched human tasks (depends on item 2)

**Wave 3 — Progress visualization and dashboard control center**
11. Extend `lib/progress-extractor.js` for wave-level fields (depends on item 2)
12. Extend `lib/push-client.js` payload with requirements + waves (depends on items 1, 2)
13. Neon DDL: add requirement_snapshots + wave_snapshots tables
14. Dashboard panels: requirements tree, wave progress bars, velocity chart (depends on items 12, 13)
15. Dashboard pages: /projects/[slug]/roadmap full control center view (depends on item 14)

## Sources

- Direct inspection of `/home/alucardmessangeroflight/.claude/plugins/seraphim/` (2026-04-09)
- `lib/roadmap.js` — atomic write pattern, nextFeatureId pattern (reference implementation for requirements.js)
- `lib/push-client.js` — fire-and-forget dashboard push pattern
- `lib/phase-state.js` — per-phase state CRUD pattern
- `lib/decisions-logger.js` — append-only JSONL pattern
- `lib/markers.js` + `commands/tasks.md` — task marker system (extend for enriched task types)
- `commands/new-milestone.md` — node -e inline lib call pattern (canonical reference)
- `.planning/seeds/SEED-001-self-improving-intelligence.md` — seed format reference
- `.planning/PROJECT.md` — v3.2 feature list and constraints

---
*Architecture research for: Seraphim v3.2 idea-to-shipped workflow integration*
*Researched: 2026-04-09*
