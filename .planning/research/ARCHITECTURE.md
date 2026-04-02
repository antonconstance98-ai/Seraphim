# Architecture Research: v1.1 Global Metrics Dashboard Integration

**Domain:** Global aggregation layer over per-project hook-generated JSONL data, with HTML dashboard output
**Researched:** 2026-04-02
**Confidence:** HIGH — based on live codebase inspection, not assumptions

---

## System Overview

```
┌──────────────────────────────────────────────────────────────────────┐
│                   CLAUDE CODE SESSION (any project)                  │
│                                                                      │
│  PostToolUse hook fires on Bash/Edit/Write                           │
│  ┌────────────────────────────────────────────────────────────┐      │
│  │ codex-token-logger.js                                       │      │
│  │   → reads stdin: { cwd, session_id, tool_result }          │      │
│  │   → detects [CODEX_RESULT] marker                          │      │
│  │   → appends JSONL record to {cwd}/.planning/token-log.jsonl│      │
│  └────────────────────────────────────────────────────────────┘      │
│                                                                      │
│  SessionStart hook fires when session begins                         │
│  ┌────────────────────────────────────────────────────────────┐      │
│  │ codex-cost-reporter.js (EXISTING — per-project scope)      │      │
│  │   → reads {cwd}/.planning/token-log.jsonl                  │      │
│  │   → writes {cwd}/.planning/session-reports/YYYY-MM-DD.md  │      │
│  │   → outputs additionalContext summary to Claude            │      │
│  └────────────────────────────────────────────────────────────┘      │
│                                                                      │
│  SessionStart hook (NEW — after cost-reporter)                       │
│  ┌────────────────────────────────────────────────────────────┐      │
│  │ codex-global-aggregator.js (NEW)                           │      │
│  │   → discovers all projects with token-log.jsonl            │      │
│  │   → merges records into ~/.claude/dashboard/global.jsonl   │      │
│  │   → calls dashboard generator                              │      │
│  │   → outputs additionalContext: "Dashboard updated"         │      │
│  └────────────────────────────────────────────────────────────┘      │
└──────────────────────────────────────────────────────────────────────┘
                               │
                               │ fs.readdirSync (project discovery)
                               │ fs.appendFileSync (dedup-merge)
                               ▼
┌──────────────────────────────────────────────────────────────────────┐
│                   ~/.claude/dashboard/                               │
│                                                                      │
│  global.jsonl          — merged, deduplicated records from all       │
│                          projects; append-only; keyed on             │
│                          (session_id + timestamp) for dedup          │
│                                                                      │
│  project-index.json    — discovery manifest: { project_path,        │
│                          project_name, last_seen, record_count }    │
│                          updated on each aggregation run            │
│                                                                      │
│  dashboard.html        — self-contained (inline CSS/JS), written    │
│                          by codex-dashboard-generator.js after      │
│                          every aggregation                           │
│                                                                      │
│  last-run.json         — { timestamp, projects_scanned,             │
│                            records_added, total_records }           │
│                          guards against double-processing            │
└──────────────────────────────────────────────────────────────────────┘
                               │
                               │ require('./codex-dashboard-generator')
                               ▼
┌──────────────────────────────────────────────────────────────────────┐
│               codex-dashboard-generator.js (NEW)                    │
│                                                                      │
│  Input:  ~/.claude/dashboard/global.jsonl                           │
│  Output: ~/.claude/dashboard/dashboard.html                         │
│                                                                      │
│  Computes:                                                           │
│    - Per-project totals (cost, savings, calls, review catch rate)   │
│    - Time series: daily cost and savings (last 30 days)             │
│    - Session history: last 20 sessions across all projects          │
│    - Global totals                                                   │
│                                                                      │
│  Writes: single HTML file with inline <style> and <script>          │
│    - Chart.js bundled inline (CDN URL with fallback, or bundled)    │
│    - No server, no dependencies, opens directly in browser          │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Component Responsibilities

| Component | Responsibility | File Location | Status |
|-----------|---------------|---------------|--------|
| `codex-token-logger.js` | Appends JSONL records to `{cwd}/.planning/token-log.jsonl` after each Codex call | `~/.claude/hooks/` | Existing — no changes needed |
| `codex-cost-reporter.js` | Per-project session report (Markdown) on SessionStart | `~/.claude/hooks/` | Existing — no changes needed |
| `codex-global-aggregator.js` | Discovers projects, merges JSONL to global store, triggers HTML generation | `~/.claude/hooks/` | New |
| `codex-dashboard-generator.js` | Reads global JSONL, computes metrics, writes self-contained HTML | `~/.claude/hooks/` | New |
| `~/.claude/dashboard/global.jsonl` | Global deduplicated record store (append-only) | `~/.claude/dashboard/` | New (auto-created) |
| `~/.claude/dashboard/project-index.json` | Discovery manifest with per-project metadata | `~/.claude/dashboard/` | New (auto-created) |
| `~/.claude/dashboard/dashboard.html` | The output: self-contained HTML dashboard | `~/.claude/dashboard/` | New (auto-generated) |

---

## Recommended File Structure

```
~/.claude/
├── hooks/
│   ├── codex-token-logger.js          # EXISTING — no changes
│   ├── codex-cost-reporter.js         # EXISTING — no changes
│   ├── codex-global-aggregator.js     # NEW — Phase 1
│   └── codex-dashboard-generator.js  # NEW — Phase 2
│
└── dashboard/                         # NEW — auto-created by aggregator
    ├── global.jsonl                   # merged records from all projects
    ├── project-index.json             # project discovery manifest
    ├── last-run.json                  # idempotency guard
    └── dashboard.html                 # output: open in browser
```

```
~/projects/<any-project>/
└── .planning/
    ├── token-log.jsonl                # EXISTING per-project source
    └── session-reports/              # EXISTING per-project reports
        └── YYYY-MM-DD.md
```

### Structure Rationale

- **`~/.claude/dashboard/` as global store:** User-scoped, not project-scoped. Survives project deletion. Parallel with how `~/.claude/settings.json` is user-scope for hooks.
- **`global.jsonl` append-only:** Preserves the same JSONL pattern as per-project logs. Supports time-series queries by timestamp field. Safe for concurrent reads from multiple scripts.
- **`project-index.json` separate from records:** Avoids scanning all records to answer "which projects exist?" Fast project list for dashboard header.
- **`dashboard.html` inline everything:** No Node.js server, no npm serve. Open directly with `xdg-open ~/.claude/dashboard/dashboard.html`. Self-contained.

---

## Data Flow: Per-Project JSONL to Global Dashboard

```
[Any Claude Code Session Start]
           │
           ▼
codex-global-aggregator.js (hook, runs on SessionStart)
           │
           ├── 1. Load project-index.json (known projects list)
           │        → { "/path/to/project": { last_seen, record_count, project_name } }
           │
           ├── 2. DISCOVER new projects
           │        Search roots: [ ~/projects, ~/gsd-workspaces, ~/agent ]
           │        Pattern: find -name "token-log.jsonl" -not -path "*/.claude/worktrees/*"
           │        Exclude: git worktrees (.claude/worktrees/) — these are ephemeral
           │        Add new discoveries to project-index.json
           │
           ├── 3. For each known project:
           │        a. Read all JSONL lines from {project}/.planning/token-log.jsonl
           │        b. Skip records already in global.jsonl
           │           (dedup key: session_id + timestamp string)
           │        c. Enrich each record with: { project_path, project_name }
           │        d. Append new records to ~/.claude/dashboard/global.jsonl
           │
           ├── 4. Write updated project-index.json
           │        (last_seen, record_count per project)
           │
           ├── 5. Write last-run.json
           │        { timestamp, projects_scanned, records_added, total_records }
           │
           └── 6. Call codex-dashboard-generator.js
                    (require() or child_process.spawnSync — same process is simpler)
                           │
                           ▼
               codex-dashboard-generator.js
                           │
                           ├── Read global.jsonl → array of records
                           │
                           ├── Compute per-project metrics
                           │     { project_name, total_cost, opus_baseline, savings,
                           │       savings_pct, call_count, review_count, catch_rate }
                           │
                           ├── Compute daily time series (last 30 days)
                           │     Group records by date(timestamp)
                           │     Sum: actual_cost, opus_baseline per day
                           │
                           ├── Build session history
                           │     Group records by session_id
                           │     Last 20 unique session_ids by timestamp
                           │     Per session: project, date, cost, savings, calls
                           │
                           └── Write dashboard.html (self-contained)
                                 <style> inline CSS
                                 <script> inline Chart.js + data
                                 No external dependencies
```

---

## Integration Points with Existing Architecture

### Integration Point 1: settings.json Hook Registration

The aggregator runs on `SessionStart`, after `codex-cost-reporter.js`. The hook array is ordered — add the new hook second:

```json
"SessionStart": [
  {
    "hooks": [
      {
        "type": "command",
        "command": "node \"/home/alucard/.claude/hooks/gsd-check-update.js\""
      },
      {
        "type": "command",
        "command": "node \"/home/alucard/.claude/hooks/codex-cost-reporter.js\"",
        "timeout": 15
      },
      {
        "type": "command",
        "command": "node \"/home/alucard/.claude/hooks/codex-global-aggregator.js\"",
        "timeout": 30
      }
    ]
  }
]
```

**Why after cost-reporter:** The cost-reporter writes the per-project Markdown report. The aggregator reads from per-project JSONL (not the Markdown report) — so order doesn't technically matter for correctness. Placing it after cost-reporter is conventional — per-project first, then global.

**Why timeout 30:** Discovery scans multiple directories. Writing HTML with Chart.js inline data is fast. 30s is ample; 15s is tight if many projects exist.

### Integration Point 2: Existing JSONL Schema (no changes needed)

The existing `token-log.jsonl` records written by `codex-token-logger.js` are the input. The schema is already stable:

```json
{
  "timestamp": "2026-04-02T22:52:30.644Z",
  "session_id": "30092b41-...",
  "model": "gpt-5.4",
  "source": "cli",
  "task_type": "review",
  "review_task_type": "feature",
  "verdict": "ALLOW",
  "block_summary": null,
  "tokens": {
    "input": 12860,
    "cached_input": 10624,
    "output": 365,
    "reasoning_output": 0
  },
  "cost_usd": 0.04908,
  "rate_limit_pct": null
}
```

The global aggregator adds two fields when copying to `global.jsonl`:
- `project_path` — absolute path to the project root
- `project_name` — derived from the last path segment (e.g., `Claude_X_Codex`)

No changes to `codex-token-logger.js` or `codex-cost-reporter.js`.

### Integration Point 3: Deduplication Strategy

The aggregator must be idempotent — SessionStart fires every time any project opens. The same records must not be added to `global.jsonl` twice.

**Dedup key:** `session_id + timestamp` combined as a string.
- `session_id` alone is not unique (some records have `session_id: null`)
- `timestamp` alone is not unique (multiple records per session)
- `session_id + timestamp` is unique for all observed records in the live JSONL

**Implementation:** On startup, the aggregator reads `global.jsonl` once and builds a `Set` of seen keys. New records from per-project logs are filtered against this set before appending.

**Edge case — null session_id:** Records with `session_id: null` exist (from `codex-multi-round-reviewer.js` calls outside a hook context). Use `null|timestamp` as key. This is still unique because the timestamp is precise to the millisecond.

### Integration Point 4: Project Discovery Scope

Two token-log.jsonl files exist on this machine today:
- `/home/alucard/projects/Claude_X_Codex/.planning/token-log.jsonl`
- `/home/alucard/projects/The-Crucible/.planning/token-log.jsonl`

Git worktrees at `b2b-sales-ops/.claude/worktrees/*/` have `.planning/` directories but no `token-log.jsonl` (confirmed by inspection). Worktrees must be excluded from discovery — they are ephemeral and their data would duplicate the parent project.

**Discovery strategy: configurable roots with sensible defaults.**

The aggregator scans a configurable list of root directories. Defaults cover all known project locations on this machine (from CLAUDE.md Key Paths + GSD workspace convention):

```javascript
// Default discovery roots — all known project locations on this machine
const DEFAULT_DISCOVERY_ROOTS = [
  path.join(os.homedir(), 'projects'),       // Primary project directory (CLAUDE.md)
  path.join(os.homedir(), 'agent'),           // Agent workspace (CLAUDE.md)
  path.join(os.homedir(), 'gsd-workspaces'),  // GSD isolated workspaces
  '/mnt/hdd'                                  // User files from Windows (CLAUDE.md)
];

// User can extend via ~/.claude/dashboard/config.json
// { "extra_roots": ["/path/to/more/projects"] }
const configPath = path.join(dashboardDir, 'config.json');
let extraRoots = [];
try {
  const cfg = JSON.parse(fs.readFileSync(configPath, 'utf8'));
  extraRoots = cfg.extra_roots || [];
} catch (e) { /* no config file — use defaults only */ }

const DISCOVERY_ROOTS = [...DEFAULT_DISCOVERY_ROOTS, ...extraRoots]
  .filter(root => fs.existsSync(root));  // Skip roots that don't exist
```

**Why configurable:** Hardcoded roots miss valid token logs if the user creates projects in new locations. The `config.json` mechanism lets users add roots without modifying hook code. Defaults cover all locations documented in CLAUDE.md.

**Exclusion patterns:** Skip any path matching these patterns:
- `/.claude/worktrees/` — ephemeral GSD worktrees (would duplicate parent project data)
- `/node_modules/` — npm packages should never contain token logs
- `/.git/` — git internals

**Note:** `~/.claude/projects/` contains Claude Code session JSONL data (not project token logs) and should NOT be scanned.

**Discovery command (Node.js):**
```javascript
const { execSync } = require('child_process');
// Use find for speed — globbing caches and npm dirs is slow
const findCmd = `find ${root} -maxdepth 5 -name "token-log.jsonl" -not -path "*/.claude/worktrees/*" -not -path "*/node_modules/*" -not -path "*/.git/*" 2>/dev/null`;
const output = execSync(findCmd, { encoding: 'utf8', stdio: ['pipe', 'pipe', 'ignore'] });
```

`-maxdepth 5` covers `projects/NAME/.planning/token-log.jsonl` (depth 3) and deeper structures like `/mnt/hdd/CATEGORY/NAME/.planning/token-log.jsonl` (depth 4) with headroom.

---

## Architectural Patterns

### Pattern 1: Require-Based Module Composition

**What:** The aggregator calls the generator via `require('./codex-dashboard-generator')`, not via `child_process.spawnSync`.

**When to use:** Both scripts run in the same SessionStart hook invocation. `require()` avoids a second Node.js process startup, shares the already-loaded JSONL data, and keeps the total SessionStart overhead under 30 seconds.

**Trade-off:** Both scripts must be in `~/.claude/hooks/` (same directory). This is already where all hook scripts live — not a constraint.

**Example:**
```javascript
// codex-global-aggregator.js
const generator = require('./codex-dashboard-generator');
// After merging records:
generator.generateDashboard(dashboardDir);
```

```javascript
// codex-dashboard-generator.js
function generateDashboard(dashboardDir) {
  const globalLog = path.join(dashboardDir, 'global.jsonl');
  // ... read, compute, write HTML
}
module.exports = { generateDashboard };
```

### Pattern 2: Append-Only Global Log with In-Memory Dedup

**What:** Never rewrite `global.jsonl`. Load existing keys into a `Set` at startup, append only truly new records.

**When to use:** Always. This preserves historical data even if a per-project `token-log.jsonl` is deleted or rotated.

**Trade-off:** `global.jsonl` grows indefinitely. At current usage rates (8-11 records per session), this is ~1KB per session. 1,000 sessions = ~1MB. Not a concern for a single-user machine.

### Pattern 3: Generator as Pure Transform (No Side Effects Outside Output Dir)

**What:** `codex-dashboard-generator.js` reads from `~/.claude/dashboard/global.jsonl` and writes only to `~/.claude/dashboard/dashboard.html`. No project directory reads. No settings writes.

**When to use:** Separation of concerns. The aggregator owns discovery and merging. The generator owns computation and rendering. This makes each testable in isolation.

### Pattern 4: Self-Contained HTML (No Server Required)

**What:** Inline Chart.js from CDN URL in a `<script>` tag. Inline all computed data as a `const DATA = {...}` JavaScript literal. No `fetch()` calls at render time.

**When to use:** Always — this is a local developer tool, not a web app. The dashboard must open from `file://` protocol without a server.

**Implementation approach:** Use Chart.js from `https://cdn.jsdelivr.net/npm/chart.js` in a `<script src>` tag — this loads from CDN when internet is available, fails gracefully (no charts, but tables still render) when offline. This is simpler than bundling 200KB of Chart.js inline.

**Trade-off:** Chart.js from CDN loads from internet on each open. For an offline machine, bundle Chart.js inline. Current machine has internet — CDN approach is fine for v1.1.

---

## New vs Modified Components

### New Components (to build)

| Component | Type | Phase |
|-----------|------|-------|
| `~/.claude/hooks/codex-global-aggregator.js` | New hook script | Phase 1 |
| `~/.claude/hooks/codex-dashboard-generator.js` | New generator module | Phase 2 |
| `~/.claude/dashboard/` directory | Auto-created by aggregator | Phase 1 |
| SessionStart hook entry in `~/.claude/settings.json` | New hook registration | Phase 1 |

### Modified Components (what changes)

| Component | Change | Why |
|-----------|--------|-----|
| `~/.claude/settings.json` | Add `codex-global-aggregator.js` to SessionStart hooks array | Register new hook |

### Unchanged Components (confirmed no modifications needed)

| Component | Why unchanged |
|-----------|---------------|
| `codex-token-logger.js` | JSONL schema already has all needed fields; no changes |
| `codex-cost-reporter.js` | Per-project report unchanged; global report is additive |
| Per-project `token-log.jsonl` files | Source data, never modified by aggregator |
| All other hook scripts | No interaction with dashboard system |

---

## Build Order

The dependency graph drives this order:

```
Phase 1: Data Pipeline — Aggregator + Storage

  Step 1. Create ~/.claude/dashboard/ directory structure
          (can be done by the hook itself on first run)

  Step 2. Implement codex-global-aggregator.js
          Dependencies: none (reads existing JSONL files)
          Validates: can scan projects, can dedup, can write global.jsonl
          Test: run standalone, verify global.jsonl created with correct records

  Step 3. Register hook in ~/.claude/settings.json
          Add to SessionStart array after codex-cost-reporter.js
          Test: open a new session, verify global.jsonl updated

Phase 2: Dashboard Generator — HTML Output

  Step 4. Implement codex-dashboard-generator.js
          Dependencies: Phase 1 (needs global.jsonl to exist with real data)
          Implements: per-project table, time series, session history
          Test: run standalone, verify dashboard.html opens in browser

  Step 5. Wire generator into aggregator
          Call generator.generateDashboard() at end of aggregation run
          Test: SessionStart → global.jsonl updated → dashboard.html regenerated

Phase 3: Polish — Chart.js Integration

  Step 6. Add Chart.js time series charts to dashboard
          Dependencies: Phase 2 HTML structure established
          Adds: line chart (daily cost/savings) using existing time series data
          Test: open dashboard.html, verify charts render with real data
```

**Why this order:**

- Aggregator before generator: the generator needs `global.jsonl` to exist with real data to test against
- Data pipeline before charts: Chart.js integration is display-only; the underlying data structure must be solid first
- Registration after implementation: avoid a broken hook in the hot path until it's tested standalone

---

## Anti-Patterns to Avoid

### Anti-Pattern 1: Rewriting global.jsonl on Every Run

**What:** Read all per-project logs, compute deduplicated set, write a fresh `global.jsonl` each time.

**Why bad:** If a per-project `token-log.jsonl` is deleted (project archived, worktree cleaned), those records disappear from the global view. The append-only pattern preserves historical records permanently regardless of source file state.

**Instead:** Read existing `global.jsonl` keys into a Set, append only new records from current per-project files.

### Anti-Pattern 2: Scanning All of ~/

**What:** `find ~ -name "token-log.jsonl"` without restricting roots.

**Why bad:** The home directory contains `.npm-global`, `.cache`, `node_modules`, plugin caches, and other large directory trees. The scan would be slow and could traverse thousands of directories.

**Instead:** Scan configurable roots with `-maxdepth 5`. Defaults cover all CLAUDE.md key paths (`~/projects`, `~/agent`, `/mnt/hdd`) plus `~/gsd-workspaces`. Users can add more via `~/.claude/dashboard/config.json`.

### Anti-Pattern 3: Including Git Worktree Token Logs

**What:** Treating `.claude/worktrees/agent-*/` `.planning/token-log.jsonl` files as separate projects.

**Why bad:** Git worktrees are ephemeral GSD parallelism environments. Their JSONL records represent the same project (b2b-sales-ops) and the same session work. Including them would duplicate records for the parent project.

**Instead:** Exclude any path matching `/.claude/worktrees/`.

### Anti-Pattern 4: Blocking SessionStart for Long Discovery

**What:** Synchronous deep directory scan on every session start, including large projects or slow filesystems.

**Why bad:** SessionStart hooks run before the session is usable. A slow aggregator delays every session open. The hook has a 30s timeout — scan must complete well within this.

**Instead:** Use `find` with `-maxdepth 4` (fast, bounded). The `project-index.json` manifest caches known projects — on subsequent runs, re-scan only to find new projects, not to re-read all records.

### Anti-Pattern 5: Generating HTML with String Concatenation of Unescaped User Data

**What:** Building the HTML by concatenating project names, file paths, and block summaries directly into the HTML string.

**Why bad:** Block summaries from `codex-review-gate.js` contain arbitrary text (backticks, quotes, angle brackets). Unescaped insertion breaks the HTML or creates XSS in a local context.

**Instead:** HTML-escape all user-originated strings before insertion. A minimal `htmlEscape()` function covering `<`, `>`, `&`, `"`, `'` is sufficient.

---

## Scalability Considerations

This is a single-user local tool. "Scale" means handling more projects and longer history, not concurrent users.

| Concern | At 5 projects (now) | At 50 projects | At 500 projects |
|---------|---------------------|---------------|-----------------|
| Discovery scan time | ~100ms | ~500ms | Could exceed 30s timeout — add depth limit and config |
| global.jsonl size | ~10KB | ~100KB | ~1MB — still fast for Node.js readline |
| Dashboard render time | <1s | ~1s | ~5s — add pagination for session history table |
| Memory during generation | ~1MB | ~10MB | ~100MB — read global.jsonl as stream, not all at once |

Current machine has 2 projects with token logs. The 50-project boundary is not near. Optimize only if performance becomes noticeable.

---

## Confidence Assessment

| Area | Confidence | Source |
|------|------------|--------|
| Hook protocol (SessionStart, additionalContext, timeout) | HIGH | Live `settings.json` + `codex-cost-reporter.js` source |
| Existing JSONL schema | HIGH | Live `token-log.jsonl` from two projects, `codex-token-logger.js` source |
| Project discovery scope | HIGH | Live `find` command confirmed 2 token logs; worktrees confirmed no token logs |
| Deduplication key (session_id + timestamp) | HIGH | Verified from live JSONL: null session_ids exist, timestamps are millisecond-precise |
| Chart.js CDN approach for HTML | MEDIUM | Standard approach for local HTML files; CDN load assumption requires internet |
| require() composition vs spawnSync | MEDIUM | Both work; require() is faster; confirmed no circular dep risk (generator doesn't require aggregator) |
| global.jsonl growth rate | HIGH | 8-11 records/session observed; arithmetic projection is reliable |

---

## Sources

- Live source: `/home/alucard/.claude/hooks/codex-cost-reporter.js` — SessionStart hook pattern, JSONL reading, report generation
- Live source: `/home/alucard/.claude/hooks/codex-token-logger.js` — JSONL record schema, `[CODEX_RESULT]` marker protocol
- Live source: `/home/alucard/.claude/settings.json` — Hook registration format, SessionStart array structure, timeout values
- Live data: `/home/alucard/projects/Claude_X_Codex/.planning/token-log.jsonl` — 11 real records, schema confirmed
- Live data: `/home/alucard/projects/The-Crucible/.planning/token-log.jsonl` — 4 real records, cross-project confirmed
- Live inspection: `find ~/projects -name "token-log.jsonl"` — 2 projects with token logs; worktrees excluded confirmed
- Previous research: `.planning/research/ARCHITECTURE.md` (v1.0) — hook protocol, component boundaries, anti-patterns

---

*Architecture research for: v1.1 Global Metrics Dashboard integration with per-project hook architecture*
*Researched: 2026-04-02*
