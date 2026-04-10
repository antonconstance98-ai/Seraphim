# Phase 33: Core Command Layer - Research

**Researched:** 2026-04-09
**Domain:** Seraphim plugin command authoring, wave-based planning algorithms, seed/requirements management
**Confidence:** HIGH

## Summary

Phase 33 builds five new Seraphim commands (`/seraphim:seed`, `/seraphim:requirements`, `/seraphim:plan`, `/seraphim:execute`, `/seraphim:discuss`) plus three new lib modules (`seed-store.js`, `requirements.js`, `wave-planner.js`). All implementation decisions are locked via CONTEXT.md — this is a pure implementation phase with no open architecture choices.

The patterns are well-established in the existing codebase. Every command follows the same `.md` frontmatter + prose instructions format used by all 30 existing commands. Every lib file follows the `roadmap.js` atomic read/write pattern. The most technically interesting piece is Kahn's algorithm in `wave-planner.js` for topological dependency resolution — this is a classical CS algorithm with a clear, well-understood implementation.

The critical path is: `lib/wave-planner.js` (Kahn's algorithm) → `/seraphim:plan` (consumes it) → `/seraphim:execute` (depends on plan output). All seed and requirements commands are independent and can be built in parallel.

**Primary recommendation:** Build the three lib files first (seed-store.js, requirements.js, wave-planner.js), then build commands in dependency order. The lib files are the foundation; commands are thin dispatch wrappers over them.

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions

**D-01:** Commands use subagent dispatch pattern — each command spawns a specialized agent, keeping command files lean (~50 lines frontmatter + prompt). Matches GSD pattern (discuss→planner, plan→planner+checker, execute→executor).

**D-02:** Commands share a common `lib/` layer — new files `lib/seed-store.js`, `lib/requirements.js`, `lib/wave-planner.js` following the `roadmap.js` read/write atomic pattern.

**D-03:** Commands discover project root via existing `config.js` pattern (reads `.seraphim/config.json`).

**D-04:** All new commands use `.md` skill format with frontmatter, matching all 30 existing commands.

**D-05:** Seeds stored as `SEED-NNN.md` files in `.planning/seeds/` with YAML frontmatter (title, status, tags, created, trigger_conditions) + freeform body. `index.jsonl` for fast lookups.

**D-06:** Requirements stored in `.seraphim/requirements.json` following `roadmap.json` atomic read/write pattern via `lib/requirements.js`. REQ-IDs as keys, categories as grouping.

**D-07:** Wave dependencies extend `roadmap.json` feature objects with `waves: [{ id, tasks: [], depends_on: [] }]`. Kahn's algorithm in `lib/wave-planner.js` resolves execution order at plan time.

**D-08:** `/seraphim:discuss` produces CONTEXT.md matching GSD discuss-phase output exactly — `<domain>`, `<decisions>`, `<code_context>`, `<specifics>`, `<deferred>` sections.

**D-09:** `/seraphim:execute` matches GSD execute-phase pattern — discover plans, group by wave, spawn parallel agents per wave, sequential between waves. Reuses `gsd-executor` agent type.

**D-10:** `/seraphim:plan` includes planner + checker verification loop with max 3 iterations (per PLAN-06). Checker is optional via config toggle.

**D-11:** Seed trigger conditions stored as YAML frontmatter `trigger_conditions:` array with keyword/scope matchers. During `/seraphim:new-milestone`, scan all seeds and surface matches. Simple string matching.

**D-12:** Commands are independent implementations that follow GSD patterns but call Seraphim lib files, not GSD tools. Avoids coupling to GSD internals.

### Claude's Discretion

Internal agent prompt structures, error handling details, and UI output formatting are at Claude's discretion.

### Deferred Ideas (OUT OF SCOPE)

None — discussion stayed within phase scope.

</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| SEED-01 | User can capture a raw idea via `/seraphim:seed` with braindump-style freeform input | Command pattern: thin frontmatter + inline instructions, calls seed-store.js |
| SEED-02 | Seeds stored in `.planning/seeds/` with SEED-NNN.md format and index.jsonl for lookups | seed-store.js: atomic write pattern from roadmap.js, JSONL append for index |
| SEED-03 | User can promote a seed to a feature with requirements via `/seraphim:promote` | Reads SEED-NNN.md, calls requirements.js to create REQ-IDs, calls roadmap.js to add feature |
| SEED-04 | Seeds have trigger conditions that auto-surface during new-milestone when scope matches | trigger_conditions: YAML array; string match at new-milestone time via seed-store.js scan |
| SEED-05 | User can capture zero-friction notes via `/seraphim:note` (one write, no questions) | Trivial: write to .planning/notes/ or .planning/seeds/ with minimal frontmatter, no interaction |
| SEED-06 | User can add structured todos via `/seraphim:add-todo` with area tagging | Append to .planning/todos.jsonl or similar; area as tag in JSONL record |
| SEED-07 | User can list and select pending todos via `/seraphim:check-todos` | Read todos.jsonl, filter by status, display, allow status update |
| REQ-01 | User can define requirements with REQ-IDs via `/seraphim:requirements` (AI suggests, human approves) | AI generates suggestion list, user reviews, requirements.js writes approved to requirements.json |
| REQ-02 | Requirements grouped by category with v1/future/out-of-scope scoping | requirements.json structure: { categories: { [cat]: [{ id, description, scope, status }] } } |
| REQ-03 | REQ traceability matrix mapping REQ-IDs to phases, features, and verification status | requirements.json field: phase, feature_id, verified; matrix rendered from that data |
| REQ-04 | `lib/requirements.js` manages REQ-ID CRUD following roadmap.js atomic write pattern | Implement readRequirements/writeRequirements/nextReqId following exact roadmap.js pattern |
| PLAN-01 | Roadmap.json extended with waves, dependency graph, and success criteria per feature | wave-planner.js reads/writes waves array on feature objects in roadmap.json |
| PLAN-02 | Dependency resolution via Kahn's algorithm in `lib/wave-planner.js` | Classical BFS topological sort; detailed implementation pattern below |
| PLAN-03 | User can generate wave-structured PLAN.md via `/seraphim:plan` with tasks and done-criteria | Command: call wave-planner.js, format output as PLAN.md with wave sections |
| PLAN-04 | User can lock implementation decisions before planning via `/seraphim:discuss` producing CONTEXT.md | CONTEXT.md XML sections matching GSD discuss-phase format exactly |
| PLAN-05 | User can surface Claude's assumptions about a phase via `/seraphim:assumptions` | Read CONTEXT.md if exists, list implicit assumptions as enumerated items for user review |
| PLAN-06 | Plan verification loop — planner + checker agents with revision (max 3 iterations) | Planner generates PLAN.md; checker subagent reviews; loop until pass or 3 iterations |
| EXEC-01 | User can execute all plans in a phase via `/seraphim:execute` with wave-based parallel execution | Discover PLAN.md files, group tasks by wave, spawn parallel subagents per wave |
| EXEC-02 | User can execute a single plan via `/seraphim:execute-plan` | Same as execute but scoped to one PLAN.md file |
| EXEC-03 | User can run all remaining phases autonomously via `/seraphim:autonomous` (discuss→plan→execute per phase) | Sequential loop over phases: for each, call discuss, plan, execute |
| EXEC-04 | User can execute small ad-hoc tasks via `/seraphim:quick` with atomic commits and state tracking | Simplified forge pattern: no phases, single task, commit after |
| EXEC-05 | User can execute trivial tasks inline via `/seraphim:fast` (no subagents, no ceremony) | Inline execution only: no lib calls, no state, just do the task |
| EXEC-06 | Wave-based parallel execution with dependency analysis and agent grouping | wave-planner.js resolveWaves(); execute command groups tasks by wave number and spawns in parallel |

</phase_requirements>

## Standard Stack

### Core (all already installed)
| Library/Tool | Version | Purpose | Why Standard |
|--------------|---------|---------|--------------|
| Node.js | v22.22.0 | lib file execution via `node -e` inline scripts | All existing hooks and lib files use Node.js |
| `fs` (stdlib) | built-in | Atomic file writes, JSONL appends | Used in roadmap.js, config.js — no deps needed |
| `path` (stdlib) | built-in | Cross-platform path construction | Used everywhere in existing lib |
| `js-yaml` | unknown | YAML frontmatter parsing for seed .md files | Check if already in plugin deps; if not, use regex or string parsing |

### No New npm Dependencies
Per project decision (STATE.md: "Zero new npm dependencies — schema and command extension only"), all lib files must use only Node.js stdlib. YAML frontmatter parsing should use a simple regex/string splitter rather than importing js-yaml.

**Installation:** None required. All code uses Node.js stdlib only.

## Architecture Patterns

### Recommended Project Structure (new files only)
```
~/.claude/plugins/seraphim/
├── commands/
│   ├── seed.md           # /seraphim:seed — SEED-01
│   ├── note.md           # /seraphim:note — SEED-05
│   ├── add-todo.md       # /seraphim:add-todo — SEED-06
│   ├── check-todos.md    # /seraphim:check-todos — SEED-07
│   ├── promote.md        # /seraphim:promote — SEED-03
│   ├── requirements.md   # /seraphim:requirements — REQ-01/02/03
│   ├── plan.md           # /seraphim:plan — PLAN-03/05/06
│   ├── discuss.md        # /seraphim:discuss — PLAN-04
│   ├── assumptions.md    # /seraphim:assumptions — PLAN-05
│   ├── execute.md        # /seraphim:execute — EXEC-01/06
│   ├── execute-plan.md   # /seraphim:execute-plan — EXEC-02
│   ├── autonomous.md     # /seraphim:autonomous — EXEC-03
│   ├── quick.md          # /seraphim:quick — EXEC-04
│   └── fast.md           # /seraphim:fast — EXEC-05
└── lib/
    ├── seed-store.js     # SEED-01/02/04/11
    ├── requirements.js   # REQ-01/02/03/04
    └── wave-planner.js   # PLAN-01/02/06
```

Data files in project:
```
.planning/
├── seeds/
│   ├── SEED-NNN.md       # Individual seed files (already exists: SEED-001)
│   └── index.jsonl       # Fast lookup index (to be created)
├── todos.jsonl           # Structured todos store
└── notes/
    └── NOTE-NNN.md       # Zero-friction notes

.seraphim/
└── requirements.json     # Requirements store (to be created)
```

### Pattern 1: Command File Structure (established)
**What:** Every command is a `.md` file with YAML frontmatter + prose instructions. Commands invoke lib via `node -e` inline scripts.
**When to use:** All 14 new commands.

```markdown
---
description: "Short description"
argument-hint: "[arg1] [--flag]"
allowed-tools: ["Read", "Write", "Bash"]
---

# /seraphim:commandname

One-line purpose statement.

## Step 1 — Resolve project root

Find the project root by walking up from the current working directory:

\`\`\`bash
PROJECT_ROOT=""
DIR="$(pwd)"
while [ "$DIR" != "/" ]; do
  if [ -f "$DIR/.seraphim/config.json" ]; then
    PROJECT_ROOT="$DIR"
    break
  fi
  DIR="$(dirname "$DIR")"
done
echo "$PROJECT_ROOT"
\`\`\`

If `PROJECT_ROOT` is empty, abort: "No Seraphim project found. Run `/seraphim:new-project` first."

## Step 2 — Main logic

\`\`\`bash
PLUGIN_ROOT="$HOME/.claude/plugins/seraphim"
node -e "
  const store = require('\${PLUGIN_ROOT}/lib/seed-store');
  // ... logic
"
\`\`\`
```

Source: existing command files (`new-milestone.md`, `forge.md`, `run.md`) — verified pattern.

### Pattern 2: Lib File Structure (atomic read/write)
**What:** Lib files export named functions, use tmp+rename for atomic writes, return empty defaults on missing files.
**When to use:** All 3 new lib files must follow this exactly.

```javascript
// Source: ~/.claude/plugins/seraphim/lib/roadmap.js (canonical reference)
'use strict';

const fs = require('fs');
const path = require('path');

function readRequirements(projectRoot) {
  const reqPath = path.join(projectRoot, '.seraphim', 'requirements.json');
  try {
    if (fs.existsSync(reqPath)) {
      const raw = fs.readFileSync(reqPath, 'utf8');
      return JSON.parse(raw);
    }
  } catch (e) {
    // Return empty on any error
  }
  return { categories: {}, reqs: {} };
}

function writeRequirements(projectRoot, data) {
  const seraphimDir = path.join(projectRoot, '.seraphim');
  if (!fs.existsSync(seraphimDir)) {
    fs.mkdirSync(seraphimDir, { recursive: true });
  }
  const reqPath = path.join(seraphimDir, 'requirements.json');
  const tmpPath = path.join(seraphimDir, 'requirements.json.tmp');
  fs.writeFileSync(tmpPath, JSON.stringify(data, null, 2), 'utf8');
  fs.renameSync(tmpPath, reqPath);
}

module.exports = { readRequirements, writeRequirements };
```

### Pattern 3: SEED-NNN.md Format
**What:** YAML frontmatter with mandatory fields + freeform markdown body.
**When to use:** Every file created by `/seraphim:seed`.

```markdown
---
id: SEED-NNN
title: "Short title"
status: dormant
tags: [tag1, tag2]
created: YYYY-MM-DD
trigger_conditions:
  - "keyword1"
  - "keyword2 keyword3"
---

# SEED-NNN: Title

Freeform body content...
```

Source: existing SEED-001-self-improving-intelligence.md — verified format.

### Pattern 4: Kahn's Algorithm for Wave Resolution
**What:** Classical topological sort via BFS. Resolves task dependencies into ordered waves where all tasks in a wave can run in parallel.
**When to use:** `wave-planner.js` `resolveWaves()` function — called by `/seraphim:plan`.

```javascript
// Source: classical CS algorithm, no library needed
function resolveWaves(tasks) {
  // tasks: [{ id, depends_on: [] }]
  // Returns: [[wave0tasks], [wave1tasks], ...]

  const inDegree = {};
  const adjList = {};  // id -> [dependents]

  for (const task of tasks) {
    inDegree[task.id] = inDegree[task.id] || 0;
    adjList[task.id] = adjList[task.id] || [];
    for (const dep of (task.depends_on || [])) {
      adjList[dep] = adjList[dep] || [];
      adjList[dep].push(task.id);
      inDegree[task.id] = (inDegree[task.id] || 0) + 1;
    }
  }

  const waves = [];
  let queue = tasks.filter(t => inDegree[t.id] === 0).map(t => t.id);

  while (queue.length > 0) {
    waves.push([...queue]);
    const nextQueue = [];
    for (const id of queue) {
      for (const dependent of (adjList[id] || [])) {
        inDegree[dependent]--;
        if (inDegree[dependent] === 0) {
          nextQueue.push(dependent);
        }
      }
    }
    queue = nextQueue;
  }

  // Detect cycles: if any task has inDegree > 0, there's a cycle
  const remaining = tasks.filter(t => inDegree[t.id] > 0);
  if (remaining.length > 0) {
    throw new Error('Circular dependency detected: ' + remaining.map(t => t.id).join(', '));
  }

  return waves;
}
```

### Pattern 5: CONTEXT.md Format (GSD compatible)
**What:** The exact XML section structure produced by GSD discuss-phase. `/seraphim:discuss` must produce this exact format for compatibility with plan-phase consumers.
**When to use:** `/seraphim:discuss` output.

```markdown
<domain>
## Phase Boundary

[What this phase builds / its scope]
</domain>

<decisions>
## Implementation Decisions

### [Decision Category]
- **D-01:** Decision text
</decisions>

<code_context>
## Existing Code Insights

### Reusable Assets
- [file path] — [what it provides]
</code_context>

<specifics>
## Specific Ideas

[User's specific preferences, or "No specific requirements"]
</specifics>

<deferred>
## Deferred Ideas

[What was deferred, or "None"]
</deferred>
```

Source: verified from GSD discuss-phase.md line 757+ and existing 33-CONTEXT.md which uses this exact format.

### Pattern 6: JSONL Append Pattern
**What:** Append a single JSON record per line to a `.jsonl` file. Used for index.jsonl (seeds) and todos.jsonl.
**When to use:** seed-store.js index writes, add-todo command.

```javascript
// No tmp+rename needed for append-only logs (same pattern as task-completions.jsonl)
function appendToIndex(indexPath, record) {
  const line = JSON.stringify(record) + '\n';
  fs.appendFileSync(indexPath, line, 'utf8');
}

function readIndex(indexPath) {
  if (!fs.existsSync(indexPath)) return [];
  return fs.readFileSync(indexPath, 'utf8')
    .split('\n')
    .filter(Boolean)
    .map(line => JSON.parse(line));
}
```

### Pattern 7: YAML Frontmatter Parsing (stdlib only, no js-yaml)
**What:** Simple string split to extract YAML frontmatter without importing js-yaml.
**When to use:** seed-store.js when reading SEED-NNN.md files.

```javascript
function parseFrontmatter(content) {
  const match = content.match(/^---\r?\n([\s\S]*?)\r?\n---\r?\n([\s\S]*)$/);
  if (!match) return { frontmatter: {}, body: content };
  
  // Minimal YAML parsing: key: value and key:\n  - item lists
  const fm = {};
  const lines = match[1].split('\n');
  let currentKey = null;
  for (const line of lines) {
    const kvMatch = line.match(/^(\w[\w_-]*)\s*:\s*(.*)$/);
    if (kvMatch) {
      currentKey = kvMatch[1];
      const val = kvMatch[2].trim();
      fm[currentKey] = val.startsWith('[') ? JSON.parse(val.replace(/'/g, '"')) : val || [];
    } else if (line.match(/^\s+-\s+(.*)/) && currentKey) {
      if (!Array.isArray(fm[currentKey])) fm[currentKey] = [];
      fm[currentKey].push(line.match(/^\s+-\s+(.*)/)[1].replace(/^["']|["']$/g, ''));
    }
  }
  return { frontmatter: fm, body: match[2] };
}
```

### Anti-Patterns to Avoid
- **Coupling to GSD internals:** D-12 — commands call Seraphim lib files only, never `require` GSD tools
- **js-yaml dependency:** Zero new npm dependencies — parse YAML with stdlib string operations
- **async file I/O in lib files:** All existing lib files use sync fs — maintain consistency
- **Skipping tmp+rename on JSON files:** requirements.json and roadmap.json mutations must use atomic write pattern
- **Making checker required by default:** D-10 says checker is optional via config toggle — gate behind config check

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Topological sort | Custom recursive DFS | Kahn's BFS algorithm (in wave-planner.js) | Kahn's handles cycles gracefully, emits wave groupings naturally, proven correct |
| Project root discovery | Custom logic | Copy the bash walk-up pattern from existing commands | All 30 commands use identical pattern — deviation introduces bugs |
| Atomic JSON writes | Direct writeFileSync | tmp + renameSync (roadmap.js pattern) | Rename is atomic on POSIX; direct write can corrupt if interrupted |
| SEED ID numbering | Timestamp-based IDs | Sequential SEED-NNN scan of existing files | Consistent with feat-NNN pattern, human-readable, no collision risk |
| Requirements.json schema | Flat list | Nested by category (matches REQ-02 grouping requirement) | Flat list requires post-hoc grouping; nested makes reads O(1) by category |
| CONTEXT.md format | Custom XML | Exact GSD format (`<domain>`, `<decisions>`, `<code_context>`, `<specifics>`, `<deferred>`) | GSD plan-phase agents parse this format — deviation breaks downstream |

**Key insight:** The established patterns in this codebase are battle-tested over 30+ commands. The biggest risk in this phase is deviation from established patterns, not finding clever new approaches.

## Common Pitfalls

### Pitfall 1: Missing SEED-NNN.md Index Sync
**What goes wrong:** A SEED file is written but `index.jsonl` is not updated. Future lookups miss the seed.
**Why it happens:** Two-step write where the second step is forgotten on error paths.
**How to avoid:** In seed-store.js, write SEED-NNN.md and append to index.jsonl in the same function — treat them as one atomic operation (write file first, then append to index).
**Warning signs:** `index.jsonl` record count < number of SEED-*.md files in directory.

### Pitfall 2: Circular Dependency in Wave Resolution
**What goes wrong:** Task A depends on B, B depends on A — Kahn's algorithm stalls with tasks remaining in the queue.
**Why it happens:** User error in task dependency specification, or a bug in how depends_on arrays are populated.
**How to avoid:** After Kahn's loop, check if any tasks still have inDegree > 0 — if so, throw a descriptive error listing the cycle participants.
**Warning signs:** resolveWaves() returns fewer waves than expected, or the returned waves don't include all tasks.

### Pitfall 3: Requirements.json Concurrent Write
**What goes wrong:** Two `/seraphim:requirements` runs overlap and the second overwrites the first.
**Why it happens:** No locking on JSON files.
**How to avoid:** The atomic tmp+rename pattern minimizes (not eliminates) the window. Document that sequential use is assumed — this is a single-user tool.
**Warning signs:** Requirements disappear between runs.

### Pitfall 4: CONTEXT.md Format Drift from GSD
**What goes wrong:** `/seraphim:discuss` produces subtly different XML tags that break GSD plan-phase consumers.
**Why it happens:** Writing the sections from memory instead of the verified format.
**How to avoid:** The command file must include the exact section tags as literal strings: `<domain>`, `<decisions>`, `<code_context>`, `<specifics>`, `<deferred>`. Test with an actual GSD plan-phase run after implementation.
**Warning signs:** GSD planner reports missing CONTEXT.md sections when running on a Seraphim-generated context file.

### Pitfall 5: Wave Planner Ignoring roadmap.json Feature Scope
**What goes wrong:** wave-planner.js generates waves for all tasks globally instead of per-feature.
**Why it happens:** D-07 says waves extend feature objects — but the function might be called without scoping to the current feature.
**How to avoid:** `resolveWaves()` should accept a feature object and operate on `feature.waves[].tasks`. The command passes the specific feature, not the whole roadmap.
**Warning signs:** Plan output contains tasks from other features.

### Pitfall 6: Seraphim Commands Installed to Wrong Location
**What goes wrong:** New command files placed in project `.claude/` instead of plugin directory.
**Why it happens:** Confusion between project-level and plugin-level commands.
**How to avoid:** All new `.md` command files go to `~/.claude/plugins/seraphim/commands/`. All lib files go to `~/.claude/plugins/seraphim/lib/`. Verify with `ls ~/.claude/plugins/seraphim/commands/` after each file is created.
**Warning signs:** `/seraphim:seed` not available in Claude Code autocomplete after creation.

### Pitfall 7: SEED-NNN Numbering Gap on Error
**What goes wrong:** A seed write fails midway, incrementing the ID counter but not creating the file. Next seed skips a number.
**Why it happens:** ID assigned before file write, write fails.
**How to avoid:** Compute the next SEED-NNN by scanning existing files at write time (not from a stored counter). Scan `.planning/seeds/` for highest SEED-NNN.md, increment.
**Warning signs:** Gaps in SEED numbering (SEED-001, SEED-003, SEED-004).

## Code Examples

### seed-store.js: nextSeedId
```javascript
// Scan directory for highest existing SEED-NNN to determine next ID
function nextSeedId(seedsDir) {
  if (!fs.existsSync(seedsDir)) return 'SEED-001';
  const files = fs.readdirSync(seedsDir);
  let max = 0;
  for (const f of files) {
    const m = f.match(/^SEED-(\d+)/);
    if (m) {
      const n = parseInt(m[1], 10);
      if (n > max) max = n;
    }
  }
  return 'SEED-' + String(max + 1).padStart(3, '0');
}
```

### requirements.js: nextReqId
```javascript
function nextReqId(requirements) {
  // requirements.reqs: { 'REQ-001': {...}, ... }
  let max = 0;
  for (const id of Object.keys(requirements.reqs || {})) {
    const m = id.match(/^REQ-(\d+)$/);
    if (m) {
      const n = parseInt(m[1], 10);
      if (n > max) max = n;
    }
  }
  return 'REQ-' + String(max + 1).padStart(3, '0');
}
```

### seed-store.js: triggerConditionMatch (for SEED-04, D-11)
```javascript
// Simple string matching — check if milestone scope text matches any trigger condition
function matchesTriggerConditions(seed, milestoneContext) {
  const conditions = seed.frontmatter.trigger_conditions || [];
  if (!conditions.length) return false;
  const ctx = milestoneContext.toLowerCase();
  return conditions.some(c => ctx.includes(c.toLowerCase()));
}

// Called from /seraphim:new-milestone scan:
function surfaceMatchingSeeds(seedsDir, milestoneContext) {
  const index = readIndex(path.join(seedsDir, 'index.jsonl'));
  return index.filter(entry => {
    const content = fs.readFileSync(path.join(seedsDir, entry.filename), 'utf8');
    const { frontmatter } = parseFrontmatter(content);
    return matchesTriggerConditions({ frontmatter }, milestoneContext);
  });
}
```

### requirements.json Schema
```json
{
  "categories": {
    "Authentication": ["REQ-001", "REQ-002"],
    "Data Layer": ["REQ-003"]
  },
  "reqs": {
    "REQ-001": {
      "id": "REQ-001",
      "description": "User can log in with email + password",
      "category": "Authentication",
      "scope": "v1",
      "status": "pending",
      "phase": null,
      "feature_id": null,
      "verified": false
    }
  }
}
```

### wave-planner.js: Feature waves structure (extends roadmap.json)
```json
{
  "id": "feat-001",
  "slug": "add-auth",
  "status": "planned",
  "waves": [
    {
      "id": "wave-1",
      "tasks": [
        { "id": "task-001", "name": "Set up DB schema", "depends_on": [], "done_criteria": "Migration runs cleanly" }
      ],
      "depends_on": []
    },
    {
      "id": "wave-2",
      "tasks": [
        { "id": "task-002", "name": "Implement login endpoint", "depends_on": ["task-001"], "done_criteria": "POST /login returns JWT" }
      ],
      "depends_on": ["wave-1"]
    }
  ]
}
```

### /seraphim:plan checker loop (D-10, PLAN-06)
```
Step N: Planner + Checker Loop (max 3 iterations)

1. Generate PLAN.md with tasks, waves, done-criteria
2. If config.plan_check === false, skip to Step N+1
3. Spawn checker subagent: "Review this PLAN.md for gaps, missing done-criteria, 
   unreachable tasks, or dependency cycles. Return PASS or list of issues."
4. If PASS, continue. If issues, revise PLAN.md and increment iteration counter.
5. If iteration >= 3, continue with current PLAN.md and warn user of unresolved issues.
```

## Environment Availability

| Dependency | Required By | Available | Version | Fallback |
|------------|------------|-----------|---------|----------|
| Node.js | All lib files | Yes | v22.22.0 | — |
| `~/.claude/plugins/seraphim/lib/roadmap.js` | wave-planner.js, seed-store.js | Yes | current | — |
| `~/.claude/plugins/seraphim/lib/config.js` | all commands | Yes | current | — |
| `.planning/seeds/` directory | seed-store.js | Yes | exists | — |
| `.seraphim/` directory in project | requirements.js | No (no .seraphim in seraphim project itself) | — | Commands must create it on first write |

**Missing dependencies with no fallback:** None — all required tools are present.

**Missing dependencies with fallback:** The `.seraphim/` directory does not exist in the seraphim project (seraphim project uses `.planning/` instead). The new commands use `config.js` which walks up to find `.seraphim/config.json`. Since the seraphim project does not have this, the project root fallback is the `.planning/` directory convention. The lib files must create `.seraphim/` if missing (already handled by `fs.mkdirSync(seraphimDir, { recursive: true })` in the roadmap.js pattern).

**Important observation:** The seraphim project itself does not have a `.seraphim/config.json` — it uses `.planning/config.json`. The new Seraphim commands will need to be tested in a project that has `.seraphim/config.json`, not in the seraphim project root itself. This is expected — the seraphim repo is the plugin source, not a Seraphim-managed project.

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Single-agent commands | Subagent dispatch pattern | v3.0 | Commands stay lean; logic in specialized agents |
| Sequential task execution | Wave-based parallel execution | v3.2 (this phase) | Independent tasks run simultaneously, cutting wall time |
| No requirements management | requirements.json with REQ-IDs | v3.2 (this phase) | Traceability from idea to implementation to verification |

## Open Questions

1. **Where do todos.jsonl and notes/ live relative to seeds/?**
   - What we know: SEED-01/02 are clear (`.planning/seeds/`). SEED-05/06/07 are notes and todos.
   - What's unclear: D-05 only specifies seeds format. Notes and todos storage paths are not locked in CONTEXT.md (Claude's discretion).
   - Recommendation: Use `.planning/notes/NOTE-NNN.md` for notes (mirrors seeds pattern) and `.planning/todos.jsonl` for todos (append-only JSONL, simpler than individual files).

2. **Does `/seraphim:autonomous` call discuss + plan + execute as subagents or inline?**
   - What we know: D-01 says commands use subagent dispatch. D-09 says execute matches GSD execute-phase pattern.
   - What's unclear: Whether autonomous invokes discuss/plan/execute as sub-commands or reimplements the logic inline.
   - Recommendation: Call each as a subagent dispatch (read the .md command file and execute it) — consistent with D-01 and the run.md pipeline orchestrator pattern.

3. **What is the requirements.json location when `.seraphim/` doesn't exist in the target project?**
   - What we know: D-06 says `.seraphim/requirements.json`. D-03 says config.js pattern for root discovery.
   - What's unclear: If a project uses `.planning/` only (like the seraphim project itself).
   - Recommendation: requirements.js creates `.seraphim/` on first write (same as roadmap.js). This is the correct pattern — projects adopt `.seraphim/` as the Seraphim data directory.

## Sources

### Primary (HIGH confidence)
- `/home/alucardmessangeroflight/.claude/plugins/seraphim/lib/roadmap.js` — canonical atomic read/write pattern
- `/home/alucardmessangeroflight/.claude/plugins/seraphim/lib/config.js` — project root discovery pattern
- `/home/alucardmessangeroflight/.claude/plugins/seraphim/commands/forge.md` — subagent dispatch + sequential execution pattern
- `/home/alucardmessangeroflight/.claude/plugins/seraphim/commands/new-milestone.md` — lean command pattern
- `/home/alucardmessangeroflight/.claude/plugins/seraphim/commands/run.md` — pipeline orchestrator pattern
- `/home/alucardmessangeroflight/.claude/get-shit-done/workflows/discuss-phase.md` — CONTEXT.md exact format (lines 757+)
- `/home/alucardmessangeroflight/projects/seraphim/.planning/seeds/SEED-001-self-improving-intelligence.md` — seed frontmatter format
- `/home/alucardmessangeroflight/projects/seraphim/.planning/phases/33-core-command-layer/33-CONTEXT.md` — locked decisions

### Secondary (MEDIUM confidence)
- Kahn's algorithm — classical BFS topological sort, well-established algorithm, no external source needed

### Tertiary (LOW confidence)
- None

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH — all libs are existing stdlib; no new packages
- Architecture: HIGH — verified against 30+ existing commands and lib files
- Kahn's algorithm: HIGH — classical CS, multiple implementations verified
- CONTEXT.md format: HIGH — verified from GSD discuss-phase.md source
- Pitfalls: HIGH — derived from existing code patterns and direct code inspection

**Research date:** 2026-04-09
**Valid until:** 2026-05-09 (stable patterns, no external dependencies)
