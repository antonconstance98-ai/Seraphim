# Phase 34: Research + Session + Navigation - Research

**Researched:** 2026-04-09
**Domain:** Seraphim plugin command authoring — research tracking, session handoff, and navigation routing
**Confidence:** HIGH

## Summary

Phase 34 builds three independent subsystems onto the Seraphim plugin: a two-gate research workflow (scope then run), an enhanced session handoff system (pause/resume redesigned around HANDOFF.json), and a set of navigation commands that route the user to the next logical action. All work happens in `~/.claude/plugins/seraphim/` following patterns established in Phase 33.

The research and session systems require one new lib file (`lib/research-tracker.js`) and two new data files (`.seraphim/research.json`, `.seraphim/HANDOFF.json`). Navigation commands are pure logic — they read existing artifacts (STATE.md, ROADMAP.md, `.planning/phases/`) and emit routing suggestions without writing new state.

The existing `pause.md` and `resume.md` commands are pipeline-scoped (they persist a `phase-id` + `pipeline-phase` pair into `phase-state.js`). Phase 34 replaces their purpose with a broader, GSD-style session handoff that captures the entire work context across all active planning artifacts. The old commands can remain as-is; new commands are added alongside them.

**Primary recommendation:** Follow the `requirements.js` atomic read/write pattern for `research-tracker.js`, follow the `discuss.md` command structure for `research-scope.md`, and use inline `node -e` calls within commands rather than spawning subagents for navigation logic.

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions

**Research System**
- D-01: Two-command gate enforced via state file. `research-scope` writes `.seraphim/research.json` entry with `status: "scoped"`. `research-run` checks for scoped status before executing. Human interrogation cannot be skipped.
- D-02: `research-scope` produces structured scope doc in research.json — topic, questions, constraints, categories. Human approves scope before AI research runs.
- D-03: `map-codebase` spawns 3-4 parallel mapper agents, each analyzing a focus area (structure, conventions, stack, concerns). Results written to `.planning/codebase/`. Matches GSD codebase-mapper pattern.
- D-04: `lib/research-tracker.js` stores state in `.seraphim/research.json` following requirements.js atomic read/write pattern. Each item: id, topic, status, scope, results.

**Session Management**
- D-05: `pause` captures HANDOFF.json (phase, plan, task position, uncommitted changes, active branch, open decisions) + `.continue-here.md` (human-readable resumption guide).
- D-06: `resume` reads HANDOFF.json, injects into Claude context, displays summary, suggests next action. Deletes handoff files after successful resume.
- D-07: `session-report` reads git log since session start, counts commits, lists files changed, estimates token spend from session JSONL. Writes to `.planning/session-reports/`.

**Navigation & Routing**
- D-08: `next` uses state-machine logic — reads STATE.md + ROADMAP.md, checks for missing artifacts (CONTEXT.md/PLAN.md/SUMMARY.md/VERIFICATION.md), routes to earliest missing artifact's command.
- D-09: `do` uses keyword matching + intent classification — scans input for action verbs (plan, execute, debug, seed, research), routes to matching command. Fallback: present top 3 matches for user selection.
- D-10: `progress` shows phase table + completion bars + next action — reads ROADMAP.md phases, calculates completion %, shows current position, suggests next command.

### Claude's Discretion

- Internal prompt structures, error messages, and display formatting at Claude's discretion.

### Deferred Ideas (OUT OF SCOPE)

None — discussion stayed within phase scope.
</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| RSRCH-01 | User can scope research focus via `/seraphim:research-scope` (human interrogation gate) | D-01, D-02: state file gate enforces two-command separation; scope doc written to research.json |
| RSRCH-02 | User can run AI research via `/seraphim:research-run` (only after scope is locked) | D-01: reads research.json, validates `status: "scoped"` before delegating to research subagent |
| RSRCH-03 | Two-command separation enforced — interrogation gate cannot be skipped | D-01: `research-run` aborts with error if no scoped entry found |
| RSRCH-04 | `lib/research-tracker.js` manages research item state and categorization | D-04: follows requirements.js atomic pattern; CRUD for research.json items |
| RSRCH-05 | User can analyze codebase structure via `/seraphim:map-codebase` with parallel mapper agents | D-03: 3-4 subagents spawned in parallel, output to `.planning/codebase/` |
| SESS-01 | User can pause work with full context handoff via `/seraphim:pause` (HANDOFF.json + .continue-here.md) | D-05: new pause command replaces pipeline-scoped pause with broader GSD-style handoff |
| SESS-02 | User can resume work from previous session via `/seraphim:resume` with context restoration | D-06: reads HANDOFF.json, injects context, deletes files after successful restore |
| SESS-03 | Session reports generated via `/seraphim:session-report` with work summary and outcomes | D-07: reads git log + session JSONL, writes to `.planning/session-reports/` |
| NAV-01 | User can auto-advance to next logical step via `/seraphim:next` (discuss→plan→execute→verify progression) | D-08: state-machine reads STATE.md + phase artifact presence |
| NAV-02 | User can route freeform text to the right command via `/seraphim:do` | D-09: keyword + intent classification, top-3 fallback |
| NAV-03 | User can check project progress and route to next action via `/seraphim:progress` | D-10: reads ROADMAP.md, renders completion bars, suggests next command |
</phase_requirements>

## Standard Stack

### Core

| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| Node.js | v22.22.0 (installed) | All lib files and inline `node -e` scripts | Existing plugin runtime; zero new deps |
| fs (built-in) | — | Atomic file read/write | Already used in requirements.js, roadmap.js |
| path (built-in) | — | Cross-platform path construction | Already used throughout lib |
| child_process (built-in) | — | Spawning parallel mapper subagents | Already used in execute.md patterns |

### No New npm Dependencies

Per project constraint (STATE.md: "Zero new npm dependencies"). All functionality uses built-in Node.js modules and existing lib files.

**Installation:** None required.

## Architecture Patterns

### Recommended Project Structure

New files this phase creates:

```
~/.claude/plugins/seraphim/
├── lib/
│   └── research-tracker.js          # new — RSRCH-04
├── commands/
│   ├── research-scope.md            # new — RSRCH-01
│   ├── research-run.md              # new — RSRCH-02
│   ├── map-codebase.md              # new — RSRCH-05
│   ├── pause.md                     # replace existing (SESS-01)
│   ├── resume.md                    # replace existing (SESS-02)
│   ├── session-report.md            # new — SESS-03
│   ├── next.md                      # new — NAV-01
│   ├── do.md                        # new — NAV-02
│   └── progress.md                  # new — NAV-03

Project runtime files (written per project):
<PROJECT_ROOT>/
├── .seraphim/
│   ├── research.json                # research item state store
│   ├── HANDOFF.json                 # session pause state
│   └── .continue-here.md           # human-readable resume guide
└── .planning/
    ├── codebase/                    # map-codebase output directory
    │   ├── structure.md
    │   ├── conventions.md
    │   ├── stack.md
    │   └── concerns.md
    └── session-reports/             # session-report output directory
        └── YYYY-MM-DD-HHMM.md
```

### Pattern 1: Atomic Read/Write for research-tracker.js

Follow `requirements.js` exactly. This pattern is the established standard across all lib files.

```javascript
// Source: ~/.claude/plugins/seraphim/lib/requirements.js (verified)
'use strict';
const fs = require('fs');
const path = require('path');

function readResearch(projectRoot) {
  const p = path.join(projectRoot, '.seraphim', 'research.json');
  try {
    if (fs.existsSync(p)) return JSON.parse(fs.readFileSync(p, 'utf8'));
  } catch (e) {}
  return { items: {} };
}

function writeResearch(projectRoot, data) {
  const dir = path.join(projectRoot, '.seraphim');
  if (!fs.existsSync(dir)) fs.mkdirSync(dir, { recursive: true });
  const p = path.join(dir, 'research.json');
  const tmp = p + '.tmp';
  fs.writeFileSync(tmp, JSON.stringify(data, null, 2), 'utf8');
  fs.renameSync(tmp, p);  // atomic
}
```

Each research item schema (from D-04):

```javascript
{
  id: 'RSRCH-001',         // auto-incremented
  topic: 'string',          // from user interrogation
  status: 'scoped' | 'running' | 'complete' | 'abandoned',
  scope: {
    questions: [],           // specific questions to answer
    constraints: [],         // what NOT to research
    categories: []           // grouping labels
  },
  results: null | 'string', // filled by research-run
  created_at: 'ISO',
  completed_at: null | 'ISO'
}
```

### Pattern 2: Two-Command Gate (RSRCH-01, RSRCH-02, RSRCH-03)

`research-scope.md` is purely interrogative — it asks the user questions, builds the scope object, writes it to research.json with `status: "scoped"`, then stops. It does NOT do any research.

`research-run.md` opens research.json, finds entries with `status: "scoped"` (or a specific ID passed as argument), verifies scope exists, then delegates to a subagent for AI research. If no scoped entry is found, it aborts with a clear error:

```
Error: No scoped research found.
Run /seraphim:research-scope first to define your research topic.
```

This gate is the core of RSRCH-03. The check lives inside `research-run.md`'s first step, before any research work begins.

### Pattern 3: HANDOFF.json Schema (SESS-01, SESS-02)

The new `pause.md` replaces the pipeline-scoped version. It captures broader GSD-level state:

```javascript
// HANDOFF.json schema
{
  "captured_at": "ISO timestamp",
  "phase": "34",                          // current phase number
  "plan": "34-01-PLAN.md",               // current plan file if mid-execution
  "task_position": "Wave 2, Task 3",      // human-readable position
  "active_branch": "master",             // git branch
  "uncommitted_changes": [               // from `git status --short`
    "M .planning/phases/34-research-session-navigation/34-PLAN.md"
  ],
  "open_decisions": [],                  // any unresolved items from STATE.md
  "next_suggested_command": "/seraphim:execute 34"
}
```

`.continue-here.md` is the human-readable version — plain English summary of where work stopped and exactly what to type to resume.

`resume.md` reads HANDOFF.json, injects all fields as context, prints the summary banner, then deletes both files. After deletion it suggests the next command from `next_suggested_command`.

### Pattern 4: Navigation State Machine (NAV-01)

`next.md` reads three sources in order:

1. `STATE.md` (frontmatter `phase:` field) to determine current phase number
2. `.planning/phases/{phase}-*/` directory to discover which artifacts exist
3. Artifact presence check: CONTEXT.md → PLAN.md → SUMMARY.md → VERIFICATION.md

Decision table:

| Condition | Suggested Command |
|-----------|------------------|
| No CONTEXT.md | `/seraphim:discuss {phase}` |
| CONTEXT.md but no PLAN.md | `/seraphim:plan {phase}` |
| PLAN.md but no SUMMARY.md | `/seraphim:execute {phase}` |
| SUMMARY.md but no VERIFICATION.md | `/seraphim:verify {phase}` |
| All artifacts present | Advance phase number, repeat check |
| No more phases | "Milestone complete — run `/seraphim:progress`" |

### Pattern 5: Keyword Router (NAV-02)

`do.md` receives freeform text and applies a keyword map to route it:

```
plan, planning, roadmap, design       → /seraphim:plan
execute, build, implement, code, run  → /seraphim:execute
debug, fix, broken, error, failing    → /seraphim:debug
research, investigate, explore, learn → /seraphim:research-scope
seed, idea, capture, note             → /seraphim:seed
progress, status, where, overview     → /seraphim:progress
```

On ambiguous input (multiple verb matches), present top 3 as numbered options. User picks. Never auto-execute.

### Pattern 6: Parallel Mapper Agents (RSRCH-05)

`map-codebase.md` dispatches 4 subagents simultaneously. Each receives a focused task and writes its output to a specific file:

| Agent | Focus Area | Output File |
|-------|------------|-------------|
| Agent 1 | Directory structure, entry points, module layout | `.planning/codebase/structure.md` |
| Agent 2 | Coding conventions, naming patterns, comment style | `.planning/codebase/conventions.md` |
| Agent 3 | Technology stack, dependencies, versions | `.planning/codebase/stack.md` |
| Agent 4 | Concerns, coupling, tech debt, risks | `.planning/codebase/concerns.md` |

After all 4 complete, write `.planning/codebase/index.md` as a summary linking to all four files.

### Pattern 7: Session Report (SESS-03)

`session-report.md` gathers data from three sources:

1. `git log --since="today"` — commits made during the session
2. `git diff --stat HEAD~N HEAD` — files changed
3. Session JSONL at `~/.claude/projects/<hash>/<session-id>.jsonl` — token spend estimate (read the most recent JSONL, sum `usage.input_tokens` + `usage.output_tokens` fields)

Output written to `.planning/session-reports/YYYY-MM-DD-HHMM.md` with sections: Summary, Commits, Files Changed, Token Estimate.

### Anti-Patterns to Avoid

- **Skipping the scope gate:** `research-run.md` must check `status === "scoped"` before doing anything else. Do not allow research-run to accept a bare topic string and bypass interrogation.
- **Overwriting HANDOFF.json silently:** If HANDOFF.json already exists (from a previous un-resumed pause), warn the user and ask to confirm overwrite. Do not silently discard the previous handoff.
- **State machine hardcoding phase numbers:** `next.md` must discover phase directories dynamically from the filesystem, not from a hardcoded list. Phase numbers change across milestones.
- **`do.md` auto-executing matched commands:** Intent classification suggests; it never runs commands directly. User must confirm.
- **Creating `.planning/codebase/` without checking:** `map-codebase.md` should check if codebase analysis already exists and warn before overwriting.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Atomic file writes | Custom write + error recovery | `tmp + rename` pattern from requirements.js | Rename is atomic on Linux; custom solutions risk partial writes |
| Research ID generation | Manual string formatting | Copy `nextReqId()` pattern from requirements.js | Consistent zero-padded IDs across the system |
| Git metadata for session report | Custom git history parser | `git log --format=...` with structured format string | Git's built-in formatting is reliable; parsing git output manually is fragile |
| Token estimation from JSONL | Custom JSONL schema inference | Read `usage` field from Claude Code session files (schema verified in CLAUDE.md) | Schema is documented and stable |
| Parallel agent dispatch | Sequential subagent loop | Multiple `Task()` calls in a single batch | Seraphim already uses parallel dispatch in execute.md; use same pattern |

## Common Pitfalls

### Pitfall 1: Existing pause.md Conflict

**What goes wrong:** New `pause.md` replaces the existing file. The old command is pipeline-scoped (requires `phase-id` + `pipeline-phase` args). Code in `resume.md` still references `phase-state.js`. If the new pause.md is written without reading the old one first, the pipeline pause behavior breaks for users mid-pipeline.

**Why it happens:** The old commands serve a different purpose (pipeline state, not GSD session state). Phase 34 supersedes them for session management but does not remove the underlying pipeline concept.

**How to avoid:** New `pause.md` should NOT require `phase-id` and `pipeline-phase` args. It operates at session level. If a user is mid-pipeline, capture that in HANDOFF.json's `plan` and `task_position` fields by reading `phase-state.js` data. The old pipeline-pause behavior can be deprecated gracefully with a note in the command.

**Warning signs:** CI or existing tests that call `/seraphim:pause 01-add-auth forge` will break if arg parsing changes.

### Pitfall 2: research.json Concurrent Access

**What goes wrong:** If a user runs `research-scope` twice quickly (or in two tabs), the second write could overwrite the first item.

**Why it happens:** The atomic rename prevents file corruption but not logical race conditions in item creation.

**How to avoid:** `nextResearchId()` reads the current file each time and scans all existing keys. This is safe for single-user CLI use. Document that concurrent use is not supported.

### Pitfall 3: Session JSONL Path Discovery

**What goes wrong:** Token estimation in `session-report.md` requires finding the current session's JSONL file at `~/.claude/projects/<hash>/<session-id>.jsonl`. The hash is derived from the project path, and session ID changes each session.

**Why it happens:** The hash and session ID are not stored anywhere accessible in the plugin.

**How to avoid:** Find the most recently modified JSONL file under `~/.claude/projects/` for the current project path hash. The hash is the project root path with `/` replaced by `-` (verified pattern from CLAUDE.md). Use `ls -t` and take the first result. Sum only records from `today` (filter by timestamp field in JSONL).

### Pitfall 4: `next.md` Phase Discovery Ambiguity

**What goes wrong:** STATE.md's `phase:` field may say `34` but the directory might be `34-research-session-navigation`. Glob matching must handle the numeric prefix pattern.

**Why it happens:** Phase directories use `{number}-{slug}` format; STATE.md stores only the number.

**How to avoid:** Glob `.planning/phases/{phase}-*/` and take the first match. If multiple matches (shouldn't happen), take the lowest numeric prefix. Document the assumption.

### Pitfall 5: `.continue-here.md` Stale After Partial Resume

**What goes wrong:** If `resume.md` crashes after reading HANDOFF.json but before deleting the files, the next session will see stale handoff data.

**Why it happens:** Read and delete are not atomic.

**How to avoid:** `resume.md` should delete HANDOFF.json and `.continue-here.md` as its first action after reading, before injecting context. This matches the existing `resume.md` pattern which clears the paused flag before delegating to run.md.

## Code Examples

### research-tracker.js: nextResearchId()

```javascript
// Pattern source: requirements.js nextReqId() — verified
function nextResearchId(data) {
  let max = 0;
  for (const id of Object.keys(data.items || {})) {
    const m = id.match(/^RSRCH-(\d+)$/);
    if (m) {
      const n = parseInt(m[1], 10);
      if (n > max) max = n;
    }
  }
  return 'RSRCH-' + String(max + 1).padStart(3, '0');
}
```

### research-scope.md: Writing scoped state

```bash
PLUGIN_ROOT="$HOME/.claude/plugins/seraphim"
node -e "
  const rt = require('${PLUGIN_ROOT}/lib/research-tracker');
  const item = rt.addResearch('${PROJECT_ROOT}', {
    topic: '${TOPIC}',
    questions: ${QUESTIONS_JSON},
    constraints: ${CONSTRAINTS_JSON},
    categories: ${CATEGORIES_JSON}
  });
  console.log(item.id);
"
```

### research-run.md: Gate check

```bash
node -e "
  const rt = require('${PLUGIN_ROOT}/lib/research-tracker');
  const data = rt.readResearch('${PROJECT_ROOT}');
  const scoped = Object.values(data.items).filter(i => i.status === 'scoped');
  if (scoped.length === 0) {
    console.error('NO_SCOPED_ITEMS');
    process.exit(1);
  }
  console.log(JSON.stringify(scoped[0]));
"
```

### next.md: Artifact presence check

```bash
PHASE_DIR=$(ls -d ${PROJECT_ROOT}/.planning/phases/${PHASE_NUM}-* 2>/dev/null | head -1)
PADDED=$(printf "%02d" ${PHASE_NUM})
for artifact in CONTEXT PLAN SUMMARY VERIFICATION; do
  if ! ls "${PHASE_DIR}/${PADDED}-${artifact}.md" 2>/dev/null | grep -q .; then
    echo "MISSING:${artifact}"
    break
  fi
done
echo "ALL_PRESENT"
```

### session-report.md: Token estimation

```bash
# Find project JSONL directory
PROJECT_HASH=$(echo "${PROJECT_ROOT}" | sed 's|/|-|g' | sed 's|^-||')
JSONL_DIR="$HOME/.claude/projects/${PROJECT_HASH}"
LATEST_JSONL=$(ls -t "${JSONL_DIR}"/*.jsonl 2>/dev/null | head -1)

# Sum today's token usage
node -e "
  const fs = require('fs');
  const today = new Date().toISOString().split('T')[0];
  const lines = fs.readFileSync('${LATEST_JSONL}', 'utf8').split('\n').filter(Boolean);
  let input = 0, output = 0;
  for (const line of lines) {
    try {
      const r = JSON.parse(line);
      if (r.timestamp && r.timestamp.startsWith(today) && r.usage) {
        input += (r.usage.input_tokens || 0);
        output += (r.usage.output_tokens || 0);
      }
    } catch(e) {}
  }
  console.log(JSON.stringify({ input_tokens: input, output_tokens: output }));
"
```

## Environment Availability

Step 2.6: SKIPPED (no external dependencies — all work uses existing Node.js runtime, built-in modules, and git which is already verified installed)

## Sources

### Primary (HIGH confidence)

- `~/.claude/plugins/seraphim/lib/requirements.js` — verified atomic read/write pattern, nextReqId, CRUD structure
- `~/.claude/plugins/seraphim/lib/roadmap.js` — verified atomic read/write pattern
- `~/.claude/plugins/seraphim/commands/pause.md` — verified existing pipeline-pause implementation
- `~/.claude/plugins/seraphim/commands/resume.md` — verified existing pipeline-resume implementation
- `~/.claude/plugins/seraphim/commands/discuss.md` — verified command structure and interrogation gate pattern
- `./CLAUDE.md` (seraphim project) — verified session JSONL path schema, zero new npm deps constraint
- `.planning/STATE.md` — verified phase 34 is next, phase 33 complete
- `34-CONTEXT.md` — verified all decisions D-01 through D-10

### Secondary (MEDIUM confidence)

- `.planning/REQUIREMENTS.md` — requirements RSRCH-01 through NAV-03 descriptions
- `.planning/config.json` — verified `nyquist_validation: false`, `commit_docs: true`, `parallelization: true`

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH — zero new dependencies, all tools verified installed
- Architecture: HIGH — all patterns directly observed in Phase 33 lib files
- Pitfalls: HIGH — derived from direct code inspection of existing pause/resume commands and requirements.js
- Research item schema: HIGH — derived directly from D-04 decision
- HANDOFF.json schema: MEDIUM — structure inferred from D-05 decision text; exact field names at Claude's discretion

**Research date:** 2026-04-09
**Valid until:** 2026-05-09 (stable plugin conventions, no external dependencies)
