# Phase 1: Core PM Primitives - Research

**Researched:** 2026-04-09
**Domain:** File-based project management layer integrated into an existing Node.js slash-command plugin
**Confidence:** HIGH

---

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions

- **D-01:** Features use both auto-generated numeric IDs (feat-NNN) AND optional human-readable slugs. Schema: `{ id: "feat-001", slug: "auth-system", name: "Authentication System", ... }`. Commands accept either ID or slug for lookup.
- **D-02:** roadmap.json lives at `.seraphim/roadmap.json` per project. Structure: `{ milestones: [{ id, version, name, status, features: [{ id, slug, name, description, status, depends_on, priority }] }] }`. Status enum: planned | in-progress | complete | blocked.
- **D-03:** Auto-complete + notify: when all 6 pipeline phases pass (Crucible success), feature auto-marks as `complete` in roadmap.json AND writes a notification task to inbox ("Feature X completed -- review results"). No manual `/seraphim:done {feature}` required for pipeline completion.
- **D-04:** `/seraphim:add-feature` uses interactive prompts (step-by-step: name, milestone, description, priority). Follows the same pattern as `/seraphim:new-project`. Accept flags when provided, prompt for missing required fields.
- **D-05:** `/seraphim:inbox` groups by project first, then by task type within each project. Format: "Project A: 2 decisions, 1 research. Project B: 1 review."
- **D-06:** WIP limit is a warning, not a block. `/seraphim:start` warns if limit exceeded but allows override. Default limit: 2. Configurable in `.seraphim/config.json`.
- **D-07:** `/seraphim:pause` extends state.json with `pm` block: `{ feature_id, milestone_version, progress: { phase, status } }`. `/seraphim:resume` reads this block to restore context. Must ship in Phase 1, not retrofitted later.
- **D-08:** PM layer NEVER gates pipeline execution. Every PM command has implicit bypass -- pipeline commands work fine without roadmap.json existing. If roadmap.json is missing, PM commands create it on first use or return empty state.
- **D-09:** Add nullable `feature_id` field to `decisions-logger.js buildRecord()`. When a feature is active (in-progress), the logger includes its ID. When no feature is active, field is null. No breaking changes to existing records.
- **D-10:** Enforced exclusions: no sprint/cycle system, no story points, no time-boxing, no drag-and-drop, no external PM sync, no Gantt charts, no time estimates.

### Claude's Discretion

- Terminal output formatting for roadmap tree and inbox display
- Exact interactive prompt flow for add-feature (which fields required vs optional)
- How milestone cost is computed and displayed in archival output

### Deferred Ideas (OUT OF SCOPE)

None -- discussion stayed within phase scope
</user_constraints>

---

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| ROAD-01 | Project-level roadmap stored in `.seraphim/roadmap.json` with milestone-to-feature hierarchy, status enum, and version tagging | D-02 schema, file-write pattern from phase-state.js |
| ROAD-02 | `/seraphim:roadmap` displays current roadmap as milestone-feature tree with statuses in terminal | Terminal tree rendering pattern (see Code Examples) |
| ROAD-03 | `/seraphim:add-feature` appends a new feature to a milestone's backlog with name, description, and priority order | D-04 interactive prompt pattern from new-project.md; atomic JSON read-modify-write |
| ROAD-04 | Milestone completion and archival freezes shipped milestones to `.seraphim/milestones/vX.Y.json` and cleans active roadmap | Write-then-rename atomic pattern; milestone cost from decisions.jsonl |
| ROAD-05 | Milestone progress percentage computed from feature statuses (complete/total) and displayed in roadmap view | Derived from roadmap.json — no extra state needed |
| ROAD-07 | Milestone cost tracking aggregates decisions.jsonl costs for all features in a milestone | decisions.jsonl JSONL read + filter by feature_id (D-09) |
| QUEUE-01 | Feature backlog with `planned` status in roadmap.json; any feature not yet started is in the backlog | D-02 status enum includes `planned` as default |
| QUEUE-02 | `/seraphim:start {feature}` moves feature from `planned` to `in-progress` and launches the six-phase pipeline | roadmap.json status update + run.md delegation |
| QUEUE-03 | WIP limit (configurable, default 2) enforced on `/seraphim:start`; warns if limit exceeded before starting | config.js `max_wip` field addition; count in-progress features |
| QUEUE-04 | Feature reordering within a milestone via command or direct JSON edit | Array splice on features[] in roadmap.json; note direct JSON edit is always valid per D-08 |
| TASK-01 | `/seraphim:inbox` aggregates all pending human tasks across all active features and projects into a unified inbox | markers.js parseMarkers on forge-log.md; group by project then type (D-05) |
| TASK-02 | Human task types: decision, research, review, validation, skills -- surfaced as type labels in task lists | Existing SERAPHIM:HUMAN_TASKS marker format; type attribute already parsed by markers.js |
| TASK-03 | `/seraphim:done {task-id}` marks a human task complete without re-running the full pipeline | Patch forge-log.md marker status in-place OR write a completion event to a sidecar |
| TASK-04 | Pipeline gates (before Envision, before Architect, after Crucible) write human tasks to forge-log.md visible in inbox | Existing pattern in tasks.md; gates must emit SERAPHIM:HUMAN_TASKS markers with type attribute |
| ARCH-01 | PM layer is read-path only -- observes pipeline execution, never gates or blocks it; every PM operation has bypass | D-08; enforce in every new command: if roadmap.json missing, return empty state, never error |
| ARCH-02 | `decisions-logger.js` extended with nullable `feature_id` field linking decisions to features | D-09; additive change to buildRecord() — existing callers unaffected |
| ARCH-03 | `/seraphim:pause` state.json extended with PM context block (feature ID, milestone, progress) for session continuity | D-07; extend pause.md Step 3 write; extend resume.md Step 3 read |
| ARCH-06 | Anti-features enforced: no sprint/story-points, no time-boxing, no drag-and-drop Kanban, no external PM tool sync | D-10; enforce in code by not implementing; enforce in command help text |
</phase_requirements>

---

## Summary

Phase 1 builds a terminal-first project management layer on top of the existing Seraphim plugin. The foundational artifact is `.seraphim/roadmap.json` -- a flat JSON file holding the milestone-feature hierarchy. All PM commands read from and write to this file; none of them gate or block the pipeline. The implementation strategy is purely additive: new slash command Markdown files, a new JSON file per project, and small extensions to three existing files (`decisions-logger.js`, `config.json`, `pause.md`/`resume.md`).

The critical path is roadmap.json -- its schema is locked by D-02 and cascades into every other command. The second most important constraint is D-08 (bypass everywhere): every command must gracefully handle a missing roadmap.json by creating it or returning empty state. Violating this creates ceremony creep (Pitfall 2 from prior research).

The human task inbox (`/seraphim:inbox`) aggregates SERAPHIM:HUMAN_TASKS markers already emitted by the pipeline's forge-log.md. This avoids a new data store: the pipeline already writes the data; inbox just scans and presents it. The pause/resume PM context (D-07) must ship in this phase -- the prior research's Pitfall 4 documents why retrofitting it later is costly.

**Primary recommendation:** Implement in this order -- (1) roadmap.json schema + CRUD helpers, (2) `/seraphim:roadmap` display, (3) `/seraphim:add-feature` + `/seraphim:start`, (4) config.js `max_wip` extension, (5) decisions-logger.js `feature_id` extension, (6) pause/resume PM block, (7) `/seraphim:inbox`, (8) `/seraphim:done`, (9) milestone archival.

---

## Standard Stack

### Core (zero new packages -- verified from STACK.md research)

| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| Node.js `fs` (sync) | v22.22.0 (installed) | All roadmap.json and config reads/writes | Synchronous writes match crash-safety pattern already used in phase-state.js |
| Node.js built-in | v22.22.0 | JSON parse/stringify, path resolution | Already the plugin runtime; no new dependency |
| `markers.js` (internal) | Current | Parse SERAPHIM:HUMAN_TASKS from forge-log.md | Already parses all marker types; `parseMarkers()` is the canonical reader |
| `config.js` (internal) | Current | Read `.seraphim/config.json` including new `max_wip` | Extends existing reader; `read()` already merges defaults |
| `phase-state.js` (internal) | Current | Read pipeline phase completion for feature auto-complete signal | `readState()` returns `completed` boolean per phase |
| `decisions-logger.js` (internal) | Current | Extend `buildRecord()` with nullable `feature_id` | Additive parameter; no caller changes needed |

### No New npm Packages

The entire phase is implemented without adding any npm dependency. Confirmed by prior STACK.md research: file I/O at this scale (hundreds of features, tens of milestones) never needs a database or ORM.

---

## Architecture Patterns

### Recommended Project Structure (new files only)

```
.seraphim/
├── roadmap.json              ← NEW: milestone-feature source of truth (D-02)
├── milestones/               ← NEW: archived milestone snapshots (ROAD-04)
│   └── v3.1.json             ←       one file per archived milestone
├── config.json               ← EXTEND: add max_wip field (D-06)
└── phases/{id}/
    └── state.json            ← EXTEND: add pm block on pause (D-07)

~/.claude/plugins/seraphim/
├── commands/
│   ├── roadmap.md            ← NEW slash command
│   ├── add-feature.md        ← NEW slash command
│   ├── start.md              ← NEW slash command
│   ├── inbox.md              ← NEW slash command
│   ├── done.md               ← NEW slash command
│   ├── close-milestone.md    ← NEW slash command
│   ├── pause.md              ← EXTEND with PM context write (Step 3)
│   └── resume.md             ← EXTEND with PM context read (Step 3)
└── lib/
    ├── decisions-logger.js   ← EXTEND buildRecord() with feature_id
    ├── config.js             ← EXTEND CONFIG_DEFAULTS with max_wip: 2
    └── roadmap.js            ← NEW helper: readRoadmap, writeRoadmap, findFeature
```

### Pattern 1: Atomic JSON Read-Modify-Write

All roadmap.json mutations follow a read-modify-write cycle. Use synchronous writes for crash safety (matches phase-state.js).

**What:** Read the full JSON, mutate the in-memory object, write atomically via temp-file rename.
**When to use:** Any command that modifies roadmap.json (add-feature, start, close-milestone, auto-complete).

```javascript
// Source: derived from phase-state.js pattern (sync write) + Linux atomic rename
const fs = require('fs');
const path = require('path');

function readRoadmap(projectRoot) {
  const p = path.join(projectRoot, '.seraphim', 'roadmap.json');
  if (!fs.existsSync(p)) {
    return { milestones: [] }; // D-08: missing file = empty state, never error
  }
  return JSON.parse(fs.readFileSync(p, 'utf8'));
}

function writeRoadmap(projectRoot, data) {
  const dir = path.join(projectRoot, '.seraphim');
  if (!fs.existsSync(dir)) fs.mkdirSync(dir, { recursive: true });
  const target = path.join(dir, 'roadmap.json');
  const tmp = target + '.tmp';
  fs.writeFileSync(tmp, JSON.stringify(data, null, 2), 'utf8');
  fs.renameSync(tmp, target); // atomic on Linux
}
```

### Pattern 2: Feature Lookup by ID or Slug (D-01)

Every command that accepts a `{feature}` argument must resolve both numeric ID and slug.

```javascript
function findFeature(roadmap, ref) {
  // ref is "feat-001" or "auth-system"
  for (const milestone of roadmap.milestones) {
    for (const feature of milestone.features) {
      if (feature.id === ref || feature.slug === ref) {
        return { feature, milestone };
      }
    }
  }
  return null;
}
```

### Pattern 3: Slash Command Structure (from existing commands)

All slash commands are Markdown files with a YAML frontmatter and numbered step instructions. They use Bash blocks for Node.js one-liners. Study `new-project.md` for the interactive prompt pattern (D-04).

Key structural rules observed from existing commands:
- Step 1: Parse arguments (extract positional args and flags)
- Step 2: Resolve project root (walk up from cwd looking for `.seraphim/config.json`)
- Step 3+: Execute with Node.js one-liners using `PLUGIN_ROOT="$HOME/.claude/plugins/seraphim"`
- Final step: Print confirmation to user

The `new-project.md` interactive prompt pattern: Claude reads the command, prompts the user for each required field in sequence ("What is the feature name?"), then runs the write script after all inputs are gathered.

### Pattern 4: SERAPHIM Marker Emission for Human Tasks (TASK-04)

Pipeline gates must emit markers into forge-log.md that `/seraphim:inbox` can parse. The `markers.js` `emitMarker()` and `parseMarkers()` functions are the canonical interface.

```javascript
// Source: ~/.claude/plugins/seraphim/lib/markers.js
const { emitMarker } = require('./markers.js');

// Gate before Envision writes this to forge-log.md:
const taskMarker = emitMarker('HUMAN_TASKS', {
  count: '1',
  feature_id: 'feat-001'
});
// Then write individual task markers:
const task = emitMarker('TASK', {
  id: 'T-001',
  type: 'decision',    // decision | research | review | validation | skills
  assignee: 'human',
  status: 'pending',
  description: 'Choose database schema for feature X'
});
```

### Pattern 5: decisions-logger.js Extension (D-09, ARCH-02)

Additive change -- add `feature_id` as an optional parameter with null default. No existing callers break.

```javascript
// Extended buildRecord() -- feature_id added as nullable
function buildRecord({ phase, model, profile, tokens_in, tokens_out, cost_usd,
                       latency_ms, outcome, retry_count = 0, loop_count = 0,
                       quality_signals = {}, feature_id = null }) {
  return {
    timestamp: new Date().toISOString(),
    phase, model, profile,
    tokens_in: tokens_in || 0,
    tokens_out: tokens_out || 0,
    cost_usd: typeof cost_usd === 'number' ? cost_usd : 0,
    latency_ms: latency_ms || 0,
    outcome, retry_count, loop_count,
    feature_id,  // null when no active feature
    quality_signals: Object.assign({
      crucible_pass_rate: null, judge_kill_rate: null,
      checkpoint_catch_rate: null, loop_trigger_reason: null
    }, quality_signals)
  };
}
```

### Pattern 6: config.js max_wip Extension (D-06, QUEUE-03)

Add `max_wip: 2` to `CONFIG_DEFAULTS`. The `read()` function already merges user config over defaults -- no other change needed.

```javascript
// In config.js CONFIG_DEFAULTS object, add:
const CONFIG_DEFAULTS = {
  // ... existing fields ...
  max_wip: 2  // WIP limit; warn (not block) when exceeded
};
```

### Pattern 7: pause.md PM Context Extension (D-07, ARCH-03)

Extend the existing Step 3 write in pause.md to include a `pm` block. The same `ps.writeState()` call is used -- just extend the state object before writing.

```javascript
// Extended pause Step 3 node script:
const ps = require('${PLUGIN_ROOT}/lib/phase-state');
const state = ps.readState('${PROJECT_ROOT}', '${PHASE_ID}');
state.paused = true;
state.paused_at = new Date().toISOString();
state.current_pipeline_phase = '${CURRENT_PIPELINE_PHASE}';
// PM context block (D-07) -- all fields optional for backward compat
state.pm = {
  feature_id: '${FEATURE_ID}',       // or null if none active
  milestone_version: '${MILESTONE}', // or null
  progress: {
    phase: '${CURRENT_PIPELINE_PHASE}',
    status: 'paused'
  }
};
ps.writeState('${PROJECT_ROOT}', '${PHASE_ID}', state);
```

resume.md Step 3 reads `state.pm` (if present) and restores PM context. If `state.pm` is absent (pre-PM sessions), resume works normally -- backward compat maintained.

### Anti-Patterns to Avoid

- **Gating pipeline commands on PM state:** `run.md`, `pause.md`, `resume.md` must never fail because roadmap.json is missing or malformed (D-08).
- **Dual-write without atomic swap:** Never write to two PM files in sequence without using the temp-file rename pattern. A crash between writes leaves inconsistent state.
- **Scanning all forge-log.md files on every inbox call:** For this phase (single project context), direct scan is fine. Cross-project scan is Phase 2 (OVER-01).
- **Feature ID auto-increment from array length:** If a feature is deleted, length-based IDs collide. Use the highest existing numeric ID + 1, or read the last ID from the array.
- **Storing WIP count as a field:** Never cache WIP count. Always compute it by counting `in-progress` features in roadmap.json at call time.

---

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Human task parsing | Custom regex for forge-log.md | `markers.js parseMarkers()` | Already handles all SERAPHIM marker types including TASK and HUMAN_TASKS |
| Config reading with defaults | Custom JSON loader | `config.js read()` | Already merges user config over CONFIG_DEFAULTS; just add `max_wip` to defaults |
| Phase state read/write | Custom JSON file operations | `phase-state.js readState/writeState` | Already handles missing files, directory creation, synchronous write |
| Atomic file write | Two-step write | `writeFileSync + renameSync` | Linux `rename()` is atomic; already used in phase-state.js pattern |
| Pipeline phase tracking | Custom completion detector | `phase-state.js readState().completed` | `completed` boolean is already written by the Crucible phase on success |

---

## Common Pitfalls

### Pitfall 1: Roadmap.json Schema Drift
**What goes wrong:** Commands start adding ad-hoc fields to roadmap.json (e.g., `last_started_at`, `pipeline_run_id`) without a central schema. Over time the file becomes unparseable by older commands.
**Why it happens:** Each new command adds a convenience field.
**How to avoid:** All roadmap.json reads and writes go through `roadmap.js` helper functions (readRoadmap/writeRoadmap). Unknown fields are preserved but never relied upon.
**Warning signs:** A command reads a field not in D-02's schema.

### Pitfall 2: Ceremony Creep via Missing Roadmap Error
**What goes wrong:** A PM command throws an error when roadmap.json doesn't exist, breaking clean-project workflows.
**Why it happens:** Defensive coding treats missing file as an error.
**How to avoid:** D-08 is law. `readRoadmap()` returns `{ milestones: [] }` when file is absent. Every command that creates data auto-creates roadmap.json on first write.
**Warning signs:** Any PM command output contains "roadmap not found" as an error (not an informational message).

### Pitfall 3: WIP Count Staleness
**What goes wrong:** WIP count is stored as a field and only updated on start/complete transitions. A crash or manual JSON edit leaves the count wrong.
**Why it happens:** Caching for performance.
**How to avoid:** Always compute WIP by counting `status === 'in-progress'` features across all milestones in roadmap.json at call time. This is O(N) over a small N -- always fast enough.

### Pitfall 4: Human Task Type Mismatch
**What goes wrong:** `/seraphim:inbox` filters on task type but pipeline gates emit a different type string than inbox expects. Tasks are silently dropped.
**Why it happens:** Pipeline gates use informal type labels; inbox expects a canonical enum.
**How to avoid:** Define the canonical type enum in one place (documentation + code): `decision | research | review | validation | skills`. Pipeline gates must use exactly these values in TASK markers.

### Pitfall 5: feature_id Propagation Gap in decisions-logger
**What goes wrong:** `buildRecord()` accepts `feature_id` but callers in pipeline commands don't pass it. All decisions record `feature_id: null` even when a feature is active.
**Why it happens:** Callers must be updated; the function signature change alone is not enough.
**How to avoid:** When `run.md` invokes pipeline phases for a feature, it must read the active `feature_id` from roadmap.json and pass it to every `buildRecord()` call. Document this in `run.md` extension notes.

### Pitfall 6: Milestone Cost Computed from All Decisions (Not Feature-Scoped)
**What goes wrong:** Milestone cost aggregation sums ALL decisions.jsonl entries for the milestone's time period, not just those tagged with the milestone's features.
**Why it happens:** Filtering by `feature_id` requires D-09 to be complete first; without it, cost falls back to time-range filtering.
**How to avoid:** Milestone archival (ROAD-04, ROAD-07) must run AFTER `feature_id` is flowing through decisions-logger. Plan the archival command as a Wave 3+ task in the plan.

---

## Code Examples

### roadmap.json - Canonical Structure (D-02)
```json
{
  "milestones": [
    {
      "id": "ms-001",
      "version": "v3.1",
      "name": "Seraphim Project Management",
      "status": "in-progress",
      "features": [
        {
          "id": "feat-001",
          "slug": "core-pm-primitives",
          "name": "Core PM Primitives",
          "description": "Roadmap, feature queue, inbox, pause/resume",
          "status": "in-progress",
          "depends_on": [],
          "priority": 1
        },
        {
          "id": "feat-002",
          "slug": "cross-project-overview",
          "name": "Cross-Project Overview",
          "description": "Overview command across all projects",
          "status": "planned",
          "depends_on": ["feat-001"],
          "priority": 2
        }
      ]
    }
  ]
}
```

### /seraphim:roadmap Terminal Output Pattern
```
═══════════════════════════════════════════════════
 SERAPHIM  Roadmap
═══════════════════════════════════════════════════
 v3.1  Seraphim Project Management  [in-progress]  66%
 ├── [●] feat-001  core-pm-primitives  in-progress
 └── [ ] feat-002  cross-project-overview  planned

Legend: [●] in-progress  [✓] complete  [ ] planned  [!] blocked
```

### /seraphim:inbox Terminal Output Pattern (D-05)
```
═══════════════════════════════════════════════════
 SERAPHIM  Inbox
═══════════════════════════════════════════════════
 seraphim  [3 tasks]
   decisions  (1)
     [ ] T-001  Choose primary color scheme for dashboard
   research   (1)
     [ ] T-002  Evaluate Drizzle ORM vs raw pg
   review     (1)
     [ ] T-003  Review feat-001 Crucible output

Run /seraphim:done {task-id} to mark a task complete.
```

### Milestone Archival - Archive File Schema
```json
{
  "version": "v3.1",
  "name": "Seraphim Project Management",
  "archived_at": "2026-04-09T12:00:00Z",
  "completion_percent": 100,
  "features": [ /* snapshot of all features at archival time */ ],
  "cost_usd": 4.23,
  "cost_breakdown": [
    { "feature_id": "feat-001", "cost_usd": 2.10 },
    { "feature_id": "feat-002", "cost_usd": 2.13 }
  ]
}
```

---

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Separate state files per concern | Single roadmap.json as source of truth | D-02 decision | Reduces split-brain; one file to read for all PM state |
| PM as a gate (blocks pipeline) | PM as read-path only (D-08) | v3.1 design | Pipeline never waits on PM; ceremony creep prevented |
| Manual feature completion | Auto-complete on Crucible pass (D-03) | v3.1 design | Zero maintenance overhead for feature lifecycle |
| No feature tracking in decisions | Nullable feature_id in decisions-logger (D-09) | v3.1 Phase 1 | Enables per-feature cost aggregation for milestone archival |

---

## Open Questions

1. **How does `/seraphim:start` invoke run.md?**
   - What we know: `run.md` is the pipeline orchestrator. `/seraphim:resume` delegates to run.md via "read run.md and follow its instructions as if the user typed `/seraphim:run ...`".
   - What's unclear: `/seraphim:start {feature}` needs to (a) update roadmap.json to `in-progress`, (b) launch the pipeline. The phase-id for `run.md` is separate from the feature-id. Does `start` derive a phase-id from the feature slug, or does it prompt for one?
   - Recommendation: Derive phase-id from the feature slug + an auto-incremented number (e.g., `feat-001-core-pm-primitives` → phase-id `01-core-pm-primitives`). Document this derivation in the start.md command.

2. **Where does `/seraphim:inbox` scan for forge-log.md files?**
   - What we know: Current tasks.md requires a phase-id argument and reads one specific forge-log.md. Inbox needs to aggregate across all active features.
   - What's unclear: For Phase 1 (single-project scope), does inbox scan all phase directories under `.seraphim/phases/`? Or does it only show tasks for `in-progress` features?
   - Recommendation: Scan all `.seraphim/phases/*/forge-log.md` files where the corresponding feature's status is `in-progress` in roadmap.json. This is O(N phases) -- acceptable.

3. **Does `/seraphim:done {task-id}` mutate forge-log.md in-place?**
   - What we know: forge-log.md is a Markdown file with SERAPHIM marker comments. Marker status is `pending`.
   - What's unclear: Updating a marker in a Markdown file requires string replacement, which is fragile. Alternatively, write a sidecar `.seraphim/task-completions.jsonl` and treat it as an override layer.
   - Recommendation: Sidecar JSONL override is safer. `done` appends `{ "task_id": "T-001", "completed_at": "..." }` to `.seraphim/task-completions.jsonl`. Inbox merges this at read time. No forge-log.md mutation needed.

---

## Environment Availability

Step 2.6: SKIPPED -- Phase 1 is code/config changes only. All runtime dependencies (Node.js v22.22.0, npm, existing plugin files) are already verified installed by prior STACK.md research.

---

## Validation Architecture

`workflow.nyquist_validation` is explicitly `false` in `.planning/config.json`. This section is skipped.

---

## Sources

### Primary (HIGH confidence)
- `~/.claude/plugins/seraphim/lib/decisions-logger.js` -- live source; buildRecord() signature verified
- `~/.claude/plugins/seraphim/lib/phase-state.js` -- live source; readState/writeState pattern verified
- `~/.claude/plugins/seraphim/lib/config.js` -- live source; CONFIG_DEFAULTS and read() merge pattern verified
- `~/.claude/plugins/seraphim/lib/markers.js` -- live source; parseMarkers() and emitMarker() verified
- `~/.claude/plugins/seraphim/commands/pause.md` -- live source; Step 3 write pattern verified
- `~/.claude/plugins/seraphim/commands/resume.md` -- live source; Step 3 read and delegation pattern verified
- `~/.claude/plugins/seraphim/commands/tasks.md` -- live source; SERAPHIM:HUMAN_TASKS parsing pattern verified
- `~/.claude/plugins/seraphim/commands/new-project.md` -- live source; interactive prompt pattern verified
- `.planning/phases/01-core-pm-primitives/01-CONTEXT.md` -- all decisions D-01 through D-10
- `.planning/research/STACK.md` -- zero new npm packages decision verified
- `.planning/research/PITFALLS.md` -- all 10 pitfalls read; critical ones mapped to Common Pitfalls section

### Secondary (MEDIUM confidence)
- `.planning/research/ARCHITECTURE.md` -- integration approach; feature-queue.jsonl schema informed roadmap.json decision
- `.planning/STATE.md` -- project history and accumulated decisions

---

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH -- all files read from live plugin source; zero new packages confirmed
- Architecture patterns: HIGH -- derived directly from existing command patterns (new-project.md, pause.md, resume.md)
- Pitfalls: HIGH -- inherited from prior PITFALLS.md research (itself HIGH confidence); supplemented with Phase 1-specific pitfalls
- Open questions: MEDIUM -- implementation details not yet decided; recommendations provided

**Research date:** 2026-04-09
**Valid until:** 2026-05-09 (stable domain; prior plugin code is the constraint, not external APIs)
