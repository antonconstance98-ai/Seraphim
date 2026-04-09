# Phase 5: Session Commands and Hook Consolidation — Research

**Researched:** 2026-04-08
**Domain:** Claude Code plugin slash commands, state serialization, atomic settings.json mutation
**Confidence:** HIGH

---

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions
None — all implementation decisions deferred to Claude (see D-01 through D-04 below).

### Claude's Discretion
- **D-01:** All implementation decisions for this phase deferred to Claude. Session commands are straightforward utility commands. Hook retirement sequence is well-defined in research (atomic swap in settings.json, archive copies preserved).
- **D-02:** `/seraphim:help` format, `/seraphim:history` display format (table vs timeline), history depth, and data scope (per-project vs global) — Claude decides based on what's most useful.
- **D-03:** Hook retirement trigger criteria — Claude decides (e.g., after Phase 4 quality gates are proven, or after N successful pipeline runs).
- **D-04:** `/seraphim:pause` and `/seraphim:resume` serialization format — Claude decides, must be compatible with phase-state.js from Phase 1.

### Deferred Ideas (OUT OF SCOPE)
None — discussion stayed within phase scope.
</user_constraints>

---

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| SESS-01 | `/seraphim:help` displays all commands, profiles, phase descriptions, and configuration options | Static markdown command file; reads profiles.json and models.json to enumerate choices |
| SESS-02 | `/seraphim:history` shows pipeline run history with per-phase costs, models used, outcomes, loop counts, and total spend | Reads decisions.jsonl (already structured); groups by run via timestamp ranges or explicit run_id field |
| SESS-03 | `/seraphim:pause` persists current pipeline state (completed phases, loop counts, partial outputs) to `.seraphim/phases/{N}/state.json` for multi-session work | phase-state.js writeState() already handles this; pause adds a `paused_at` and `current_pipeline_phase` field |
| SESS-04 | `/seraphim:resume {N}` restores pipeline state and continues from where it was paused | Reads state.json `current_pipeline_phase`, delegates to run.md with `--from {phase}` |
| SESS-05 | `/seraphim:status` shows active profile, current phase progress, overrides, and model availability | Reads config.js, phase-state.js; calls each executor's `available()` via dispatch.js CLI |
| HOOK-01 | Seven redundant v2.0 hooks retired atomically from `~/.claude/settings.json` | Atomic JSON read→modify→write-temp→rename; exactly 7 entries to remove verified against live settings.json |
| HOOK-02 | Plugin hooks (`session-start.js`, `token-logger.js`) registered in same settings.json edit that removes old hooks — never both active simultaneously | Both plugin hooks already exist as files; only settings.json registration is missing |
| HOOK-03 | Archive copies of retired hooks preserved for rollback | .backup files already exist (e.g., `codex-review-gate.js.backup-2026-04-04T18-13-15-418Z`) — rollback path must be documented |
</phase_requirements>

---

## Summary

Phase 5 delivers five utility slash commands (`/seraphim:help`, `/seraphim:history`, `/seraphim:pause`, `/seraphim:resume`, `/seraphim:status`) and an atomic hook retirement that replaces seven legacy `~/.claude/settings.json` entries with the two plugin-managed hooks. All infrastructure these commands depend on already exists: `phase-state.js`, `decisions-logger.js`, `config.js`, `pricing.js`, `banner.js`, and all executors were built in Phases 1–4.

The commands are pure read/display operations except pause/resume, which extend phase-state.js's existing serialization. The hook retirement is the highest-risk task: it mutates `~/.claude/settings.json` atomically and must leave no window where both old and new hooks are active simultaneously.

**Primary recommendation:** Build the five session commands as `.md` files following the existing command pattern; implement hook retirement as a single Node.js script invoked once from a command file, using read→modify→write-temp→rename to guarantee atomicity.

---

## Standard Stack

### Core
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| Node.js built-in `fs` | v22.22.0 (installed) | Atomic file writes (`writeFileSync` to temp + `renameSync`) | Already the pattern throughout the plugin; rename is atomic on Linux |
| Node.js built-in `path`, `os` | v22.22.0 | Path construction, home directory resolution | No dependencies needed |

### Already-Built Dependencies (Reuse, Not Rebuild)
| Module | Location | Used By |
|--------|----------|---------|
| `phase-state.js` | `~/.claude/plugins/seraphim/lib/phase-state.js` | `/seraphim:pause`, `/seraphim:resume` |
| `config.js` | `~/.claude/plugins/seraphim/lib/config.js` | `/seraphim:status`, `/seraphim:help` |
| `decisions-logger.js` | `~/.claude/plugins/seraphim/lib/decisions-logger.js` | `/seraphim:history` (schema reference) |
| `banner.js` | `~/.claude/plugins/seraphim/lib/banner.js` | Output formatting for all commands |
| `dispatch.js` | `~/.claude/plugins/seraphim/executors/dispatch.js` | `/seraphim:status` executor availability probes |
| `profiles.json` | `~/.claude/plugins/seraphim/config/profiles.json` | `/seraphim:help`, `/seraphim:status` |
| `models.json` | `~/.claude/plugins/seraphim/config/models.json` | `/seraphim:help`, `/seraphim:status` |

**Installation:** No new packages required. All dependencies are built-in Node.js or already-installed plugin modules.

---

## Architecture Patterns

### Recommended Project Structure (new files only)

```
~/.claude/plugins/seraphim/
├── commands/
│   ├── help.md          # SESS-01 — new
│   ├── history.md       # SESS-02 — new
│   ├── pause.md         # SESS-03 — new
│   ├── resume.md        # SESS-04 — new
│   └── status.md        # SESS-05 — new
└── tools/
    └── retire-hooks.js  # HOOK-01/02/03 — new, invoked once from status or standalone
```

### Pattern 1: Command File Structure (md frontmatter + prompt)

All commands follow the same structure as the existing `run.md`, `forge.md`, etc.:

```markdown
---
description: "Short one-line description"
argument-hint: "[optional-args]"
allowed-tools: ["Read", "Bash"]
---

You are running /seraphim:{command}. Your job is ...
```

The command file is a prompt that Claude reads and acts on. All logic is bash node calls.

### Pattern 2: phase-state.js Pause Extension

`phase-state.js` stores state at `.seraphim/phases/{N}/state.json`. The pause format extends the existing schema with two new fields, preserving backward compatibility:

```json
{
  "phaseId": "01-add-auth",
  "loops": { "judge_envision": 0, "crucible_forge": 0 },
  "retries": {},
  "completed": false,
  "paused": true,
  "paused_at": "2026-04-08T22:00:00.000Z",
  "current_pipeline_phase": "forge"
}
```

`/seraphim:resume {N}` reads `current_pipeline_phase` from state.json and delegates to `run.md` with `--from {current_pipeline_phase}`. This reuses the existing `--from` logic built in Phase 3 — no new orchestration code needed.

### Pattern 3: decisions.jsonl History Reading

`decisions.jsonl` is append-only JSONL at `.seraphim/decisions.jsonl`. Each line has:
```json
{
  "timestamp": "...",
  "phase": "discover",
  "model": "perplexity-sonar",
  "profile": "balanced",
  "tokens_in": 1200,
  "tokens_out": 800,
  "cost_usd": 0.0024,
  "latency_ms": 3200,
  "outcome": "success",
  "retry_count": 0,
  "loop_count": 0,
  "quality_signals": { ... }
}
```

`/seraphim:history` groups records into runs. Since there is no explicit run_id field in the current schema, runs are grouped by time proximity (records within a contiguous time window for the same phase-id) or by reading phase-state.json `completed_at` timestamps. Recommended approach: group by `phase` transitions — a new "discover" entry after a "crucible" marks a new run.

### Pattern 4: Atomic settings.json Hook Retirement

The atomic sequence (verified against existing `codex-cost-reporter.js` and ARCHITECTURE.md):

```javascript
// Source: established pattern in ~/.claude hooks + Linux atomic rename guarantee
const settingsPath = path.join(os.homedir(), '.claude', 'settings.json');
const tmpPath = settingsPath + '.tmp.' + Date.now();

const raw = fs.readFileSync(settingsPath, 'utf8');
const settings = JSON.parse(raw);

// Remove 7 legacy hook entries from all event arrays
const RETIRE_COMMANDS = [
  'codex-review-gate.js',
  'codex-plan-reviewer.js',
  'codex-multi-round-reviewer.js',
  'minimax-post-scan.js',
  'minimax-compress.js',
  'codex-router.js',
  'codex-wave-validator.js'
];

for (const [event, groups] of Object.entries(settings.hooks || {})) {
  for (const group of groups) {
    group.hooks = (group.hooks || []).filter(h => 
      !RETIRE_COMMANDS.some(name => (h.command || '').includes(name))
    );
  }
  // Remove empty groups
  settings.hooks[event] = groups.filter(g => (g.hooks || []).length > 0);
}

// Add plugin hooks to SessionStart and PostToolUse (if not already present)
// ...

fs.writeFileSync(tmpPath, JSON.stringify(settings, null, 2), 'utf8');
fs.renameSync(tmpPath, settingsPath);  // atomic on Linux
```

### Pattern 5: Executor Availability Probe for /seraphim:status

Each executor exposes `available()`. The status command calls them via bash node inline:

```bash
PLUGIN_ROOT="$HOME/.claude/plugins/seraphim"
node -e "
  const dispatch = require('${PLUGIN_ROOT}/executors/dispatch.js');
  // Or call each executor directly:
  const executors = ['codex-exec', 'minimax-exec', 'gemini-exec', 'qwen-exec', 'perplexity-exec'];
  Promise.all(executors.map(async name => {
    const exec = require(\`${PLUGIN_ROOT}/executors/\${name}\`);
    const avail = await exec.available();
    return { name, available: avail };
  })).then(results => console.log(JSON.stringify(results)));
"
```

### Anti-Patterns to Avoid

- **Never both old and new hooks active simultaneously:** The retirement script must add plugin hook entries in the same write as it removes legacy entries. Two separate writes create a window where hooks are missing entirely or doubled.
- **Never rely on shell history format for /seraphim:history:** Parse decisions.jsonl programmatically. Grep-based parsing breaks on JSON with embedded newlines.
- **Never build a custom state format for pause/resume:** Extend the existing phase-state.js schema. A second state file creates sync issues.
- **Never use async hooks for context injection:** The plugin's session-start.js and token-logger.js are synchronous (they write to stdout before exit). Keep them that way.

---

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Atomic file write | Custom lock file + write | `writeFileSync(tmp) + renameSync` | rename is atomic on Linux; lock files can be left behind on crash |
| Cost aggregation for history | Custom token counter | Read decisions.jsonl `cost_usd` field | decisions-logger.js already records per-call costs in Phase 4 |
| Executor availability | HTTP health ping per service | Call `executor.available()` on each exec module | already implements correct availability semantics (inference probe, not just port check) |
| Run grouping for history | Custom run_id generation | Group by time proximity + phase ordering in decisions.jsonl | No schema change needed; phase sequence detect is sufficient for v3.0 |

---

## Runtime State Inventory

> Included because HOOK-01/02/03 are a migration/retirement phase.

| Category | Items Found | Action Required |
|----------|-------------|-----------------|
| Stored data | decisions.jsonl has no run_id field | Code-only: history grouping logic reads time proximity; no migration |
| Live service config | `~/.claude/settings.json` — 7 entries across PostToolUse, Stop, SubagentStop, PreToolUse events | Atomic retirement script removes all 7 in one write |
| OS-registered state | No OS-level registrations (hooks are not systemd/cron entries) | None |
| Secrets/env vars | No env vars reference old hook names | None |
| Build artifacts | `.backup` files already exist for all 7 hooks in `~/.claude/hooks/` (e.g., `codex-review-gate.js.backup-2026-04-04T18-13-15-418Z`) | Move `.backup` files to `~/.claude/hooks/archive/` directory for clean rollback path |

**Hook locations verified in live settings.json:**
- `PostToolUse`: codex-wave-validator.js, minimax-post-scan.js, minimax-compress.js
- `Stop`: codex-review-gate.js
- `SubagentStop`: codex-plan-reviewer.js (also codex-superpowers-plan-reviewer.js — NOT a retirement target)
- `PreToolUse`: codex-router.js
- **Not found in settings.json:** codex-multi-round-reviewer.js — verify before retirement script runs

**HOOK-02 gap:** Plugin hooks (`session-start.js`, `token-logger.js`) exist as files but are NOT currently registered in `~/.claude/settings.json`. The retirement script must add them in the same atomic write.

---

## Common Pitfalls

### Pitfall 1: Retiring codex-multi-round-reviewer.js When It May Not Be Registered
**What goes wrong:** The retirement script tries to remove a hook entry that isn't there, logs a false success, then the overall retirement appears complete but counts don't match.
**Why it happens:** The live settings.json (inspected 2026-04-08) does not show codex-multi-round-reviewer.js in any event. It may never have been registered, or was previously removed.
**How to avoid:** The retirement script logs how many entries it found and removed for each of the 7 targets. Zero removed for a target is logged explicitly (not silently skipped).
**Warning signs:** The summary shows "6 hooks removed" rather than "7 hooks removed" — expected if multi-round-reviewer was absent.

### Pitfall 2: Double-Registering Plugin Hooks
**What goes wrong:** plugin hooks (session-start.js, token-logger.js) are added to settings.json even though they are already there from a previous partial attempt.
**Why it happens:** The retirement script doesn't check before adding.
**How to avoid:** Before adding plugin hook entries, filter existing entries for that command path. Only add if absent.
**Warning signs:** Session start prints two Seraphim banner lines.

### Pitfall 3: /seraphim:resume Without Paused State
**What goes wrong:** User runs `/seraphim:resume 01-add-auth` on a phase that was never paused — state.json exists but has no `paused: true` field. The command falls through to run from the beginning or fails silently.
**How to avoid:** Resume command checks `state.paused === true` and `state.current_pipeline_phase` is set. If not, print clear error: "Phase {N} is not paused. Use `/seraphim:run {N}` to start fresh."

### Pitfall 4: /seraphim:history on Empty decisions.jsonl
**What goes wrong:** File doesn't exist or is empty; JSON parse fails with cryptic error.
**How to avoid:** Check file existence before reading. On empty/missing, print "No pipeline runs recorded yet."

### Pitfall 5: settings.json Corrupt on Failed Write
**What goes wrong:** Node process crashes mid-write. Without the tmp+rename pattern, settings.json ends up as a partial JSON document. Claude Code fails to start next session.
**Why it happens:** Developers use `fs.writeFileSync(settingsPath, ...)` directly.
**How to avoid:** Always write to `settingsPath + '.tmp.' + Date.now()` then `fs.renameSync(tmpPath, settingsPath)`.

---

## Code Examples

### Pause State Write (extends phase-state.js schema)
```javascript
// In pause.md bash node call:
const ps = require(`${pluginRoot}/lib/phase-state`);
const state = ps.readState(projectRoot, phaseId);
state.paused = true;
state.paused_at = new Date().toISOString();
state.current_pipeline_phase = currentPhase; // e.g. "forge"
ps.writeState(projectRoot, phaseId, state);
```

### Resume Delegation (reuse run.md --from logic)
```markdown
## Step N — Resume from paused phase
Read state.json. If `paused: true`, set `current_pipeline_phase`. Then follow run.md
instructions with `--from {current_pipeline_phase}` exactly as if the user typed:
  `/seraphim:run {phaseId} --from {current_pipeline_phase}`
Clear `paused` and `paused_at` fields before proceeding.
```

### History Grouping Logic
```javascript
// Group decisions.jsonl into runs by detecting "discover" phase starting a new sequence
const lines = fs.readFileSync(decisionsPath, 'utf8').trim().split('\n');
const records = lines.map(l => JSON.parse(l));
const runs = [];
let currentRun = [];
for (const rec of records) {
  if (rec.phase === 'discover' && currentRun.length > 0) {
    runs.push(currentRun);
    currentRun = [];
  }
  currentRun.push(rec);
}
if (currentRun.length > 0) runs.push(currentRun);
```

### Atomic Hook Retirement
```javascript
// retire-hooks.js — invoked once
const settingsPath = path.join(os.homedir(), '.claude', 'settings.json');
const tmpPath = settingsPath + '.tmp.' + Date.now();
const settings = JSON.parse(fs.readFileSync(settingsPath, 'utf8'));

const RETIRE = ['codex-review-gate.js','codex-plan-reviewer.js',
  'codex-multi-round-reviewer.js','minimax-post-scan.js',
  'minimax-compress.js','codex-router.js','codex-wave-validator.js'];

let removedCount = 0;
for (const groups of Object.values(settings.hooks || {})) {
  for (const group of groups) {
    const before = (group.hooks || []).length;
    group.hooks = (group.hooks || []).filter(h => 
      !RETIRE.some(n => (h.command || '').includes(n))
    );
    removedCount += before - group.hooks.length;
  }
}

// Add plugin hooks if absent
// ... (SessionStart and PostToolUse entries)

fs.writeFileSync(tmpPath, JSON.stringify(settings, null, 2), 'utf8');
fs.renameSync(tmpPath, settingsPath);
console.log(`Retired ${removedCount} legacy hook entries. Plugin hooks registered.`);
```

---

## Environment Availability

| Dependency | Required By | Available | Version | Fallback |
|------------|------------|-----------|---------|----------|
| Node.js | All tools/retire-hooks.js | ✓ | v22.22.0 | — |
| `~/.claude/settings.json` | HOOK-01/02/03 | ✓ | verified 2026-04-08 | — |
| `.seraphim/decisions.jsonl` | SESS-02 history | varies per project | — | Print "No history yet" |
| Plugin hook files | HOOK-02 | ✓ | session-start.js, token-logger.js present at `~/.claude/plugins/seraphim/hooks/` | — |
| Legacy hook .backup files | HOOK-03 | ✓ | 8 .backup files exist in `~/.claude/hooks/` | — |

**Missing dependencies with no fallback:** None.

**Missing dependencies with fallback:**
- decisions.jsonl: may not exist in fresh projects — history command must handle gracefully.

---

## Validation Architecture

Tests are not defined for this phase (utility commands + config mutation). Validation is manual smoke-test based:

| Req ID | Behavior | Test Type | Command | Notes |
|--------|----------|-----------|---------|-------|
| SESS-01 | `/seraphim:help` prints all commands | Manual smoke | `/seraphim:help` | Verify table lists 11 commands |
| SESS-02 | `/seraphim:history` parses decisions.jsonl | Manual smoke | `/seraphim:history` in a project with runs | Verify run grouping, cost totals |
| SESS-03 | `/seraphim:pause` writes paused:true to state.json | Manual + file check | `/seraphim:pause 01-test` then `cat .seraphim/phases/01-test/state.json` | |
| SESS-04 | `/seraphim:resume` clears paused flag and continues | Manual smoke | `/seraphim:resume 01-test` | Verify --from delegation to run.md |
| SESS-05 | `/seraphim:status` shows profile + availability | Manual smoke | `/seraphim:status` | Verify per-executor available/unavailable |
| HOOK-01/02 | No legacy hooks fire; plugin hooks fire | Automated check | `grep codex-review-gate ~/.claude/settings.json` should return nothing | |
| HOOK-03 | Archive copies exist | File check | `ls ~/.claude/hooks/archive/` | 7+ files expected |

**Phase gate:** Before marking phase complete, run the HOOK-01 grep check programmatically to confirm zero legacy entries remain.

---

## Open Questions

1. **codex-multi-round-reviewer.js not in settings.json**
   - What we know: Live settings.json (2026-04-08) does not contain this hook in any event group
   - What's unclear: Was it removed in a prior cleanup, or never registered?
   - Recommendation: retirement script logs "0 removed for codex-multi-round-reviewer.js" explicitly; treat as expected, not an error

2. **codex-superpowers-plan-reviewer.js in SubagentStop**
   - What we know: This hook exists in SubagentStop and is NOT in the retirement list (CONTEXT.md lists only 7 targets)
   - What's unclear: Should it be retired alongside codex-plan-reviewer.js?
   - Recommendation: Leave it; it is not in the retirement list. The plan should explicitly note it is preserved.

3. **run_id absent from decisions.jsonl**
   - What we know: decisions-logger.js buildRecord() does not include a run_id field
   - What's unclear: Is time-proximity grouping sufficient for history display, or should a run_id be added to decisions-logger?
   - Recommendation: Time-proximity grouping is sufficient for v3.0 history. Adding run_id would require a schema change and is Phase 6 adaptive intelligence territory.

---

## Sources

### Primary (HIGH confidence)
- Direct inspection of `~/.claude/settings.json` (2026-04-08) — verified live hook entries and event groupings
- Direct inspection of `~/.claude/plugins/seraphim/lib/phase-state.js` — verified state schema and writeState API
- Direct inspection of `~/.claude/plugins/seraphim/lib/decisions-logger.js` — verified REQUIRED_FIELDS and record schema
- Direct inspection of `~/.claude/plugins/seraphim/lib/config.js` — verified config read/write API
- Direct inspection of `~/.claude/plugins/seraphim/commands/run.md` — verified `--from` delegation pattern
- `.planning/research/PITFALLS.md` — Pitfall 2 (hook double-registration), atomic retirement pattern
- `.planning/research/ARCHITECTURE.md` — component responsibilities and file locations

### Secondary (MEDIUM confidence)
- `~/.claude/hooks/codex-cost-reporter.js` — session reporting patterns (JSONL reading, cost aggregation)

---

## Metadata

**Confidence breakdown:**
- Session commands (SESS-01 to SESS-05): HIGH — all data sources and patterns verified in live code
- Hook retirement (HOOK-01 to HOOK-03): HIGH — live settings.json inspected; atomic write pattern established
- History run grouping: MEDIUM — time-proximity heuristic works for v3.0 but is not schema-guaranteed

**Research date:** 2026-04-08
**Valid until:** 2026-05-08 (stable; no fast-moving dependencies)
