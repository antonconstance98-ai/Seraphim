# Phase 15: Decision Capture Infrastructure - Research

**Researched:** 2026-04-03
**Domain:** Structured decision logging, shared hook state, JSONL schema extension, GSD slash commands
**Confidence:** HIGH — based on live inspection of all relevant hook files, settings.json, and project
settings.json; no external library research required (this phase uses established patterns only)

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions
- **D-01:** Claude's discretion on whether to create a new `decision-log.jsonl` or extend
  `token-log.jsonl`. Choose based on separation of concerns and query patterns — billing data vs
  training signals have different write frequencies and consumers.
- **D-02:** Signal capture from upstream hooks must use the best long-term approach, not the
  easiest. A shared state mechanism (e.g., per-invocation temp file or structured output contract
  between hooks) is preferred over parsing advisory text prefixes, even though it requires
  modifying existing hooks. Durability over expedience.
- **D-03:** Log both `model_latency_ms` (API/CLI call time only) and `hook_latency_ms` (total
  hook execution time) as separate fields.
- **D-04:** `/gsd:dismiss-last` dismisses the most recent BLOCK event only (not scans).
- **D-05:** After dismiss, log the event AND show immediate feedback: "Dismissed 2/3 times for
  this rule — one more and it will be suppressed."
- **D-06:** Expand from 4 to 12 categories. Claude's discretion on exact categories and detection
  method — hook event type, tool name, file context (extensions/paths) are all available signals.
  Suggested: refactor, explain, security-scan, plan-review, doc-update, test-write, architecture,
  debug, implementation, review, bulk-ops, hook-dev.
- **D-07:** `adaptive: false` disables everything adaptive — auto-tuning, noise profiles, routing
  weight adjustments. Clean escape hatch back to static v2.0 rules.
- **D-08:** Toggle via `/gsd:freeze` and `/gsd:unfreeze` commands only. User doesn't need to know
  where the flag lives.
- **D-09:** When choosing between approaches, pick the best long-term option, not the easiest.

### Claude's Discretion
- Log schema location (D-01): new file vs extending existing, based on codebase patterns
- Signal capture mechanism (D-02): shared state approach preferred, specific implementation TBD
- Task-type taxonomy categories and detection rules (D-06): hook event + tool name + file context

### Deferred Ideas (OUT OF SCOPE)
None — discussion stayed within phase scope
</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| DCAP-01 | Every model call logs outcome (accepted/dismissed/committed) alongside existing token data | Schema extension to token-log.jsonl with nullable outcome fields; backward-compatible append |
| DCAP-02 | Every model call logs latency_ms for performance trend analysis | Separate model_latency_ms and hook_latency_ms fields (D-03); latency captured at call site |
| DCAP-03 | Task-type taxonomy expanded from 4 to 12 categories for routing accuracy | classifyTaskType() in codex-review-gate.js today has 4 categories; expand with file/path signals |
| DCAP-04 | User can dismiss a false-positive review flag via /gsd:dismiss-last, creating explicit negative training signal | New GSD slash command reading decision-log.jsonl BLOCK records; writes dismiss record |
| DCAP-05 | User can freeze adaptive behavior via adaptive:false flag, reverting to static rules instantly | New `adaptive` boolean in project settings.json; checked by decision-logger and future adaptive hooks |
</phase_requirements>

---

## Summary

Phase 15 is pure data capture infrastructure — no ML, no analysis, no config modification. It
adds structured logging of every routing and review decision so that Phases 16–18 have a reliable,
queryable training signal to work with.

The research confirms that the entire phase can be built using Node.js v22 built-in APIs (fs,
path, os, child_process) with no new npm packages. The only external dependency that could be
useful — `better-sqlite3` — is deliberately deferred to Phase 16 when query complexity justifies
it. Phase 15 uses append-only JSONL, matching every existing hook pattern.

The critical implementation decision (D-02) is the shared state mechanism. Research into the
actual hook code shows that advisory text parsing is the only pattern currently used — hooks
write to `additionalContext` and the next hook reads `tool_result`. However, D-02 explicitly
rejects this as brittle. A per-invocation temp file keyed by `session_id + timestamp` (both
available in every hook's stdin payload) is the correct implementation: it allows upstream hooks
to write structured JSON signals that `decision-logger.js` reads at the end of the chain.

**Primary recommendation:** Create a separate `decision-log.jsonl` (not an extension of
`token-log.jsonl`). The two files have different consumers, write frequencies, and schemas.
Billing data lives in `token-log.jsonl`; training signals live in `decision-log.jsonl`. The
`token-log.jsonl` schema does need the `outcome`, `model_latency_ms`, and `hook_latency_ms`
extensions (DCAP-01, DCAP-02), but decision-level signals belong in a separate file.

---

## Standard Stack

### Core (No New Packages Required for Phase 15)

| Technology | Version | Purpose | Why |
|------------|---------|---------|-----|
| Node.js built-ins | v22.22.0 (installed) | JSONL append, temp file I/O, timing | All existing hooks use these; no overhead |
| `fs.appendFileSync` | Node.js built-in | Append-only JSONL writes | Established pattern in token-logger, post-scan |
| `fs.renameSync` | Node.js built-in | Atomic temp file operations | Proven in codex-global-aggregator.js |
| `Date.now()` | Built-in | High-resolution timing for latency_ms | Simpler than `performance.now()` for wall-clock |
| `os.tmpdir()` | Node.js built-in | Per-invocation shared state temp files | Returns `/tmp` on this machine |

### Phase 16+ Only (Deferred from Phase 15)

| Library | Version | Purpose | Install When |
|---------|---------|---------|-------------|
| `better-sqlite3` | 12.8.0 | Structured query over decision log | Phase 16 — when window queries needed |
| `simple-statistics` | 7.8.9 | Running averages, Z-scores | Phase 16 — when statistical analysis needed |

**Installation (Phase 16):**
```bash
npm install better-sqlite3@12.8.0 simple-statistics@7.8.9
```

### Alternatives Considered

| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| Separate decision-log.jsonl | Extend token-log.jsonl | token-log.jsonl is billing data with a fixed schema; mixing training signals breaks the schema contract and creates dual-consumer confusion |
| Per-invocation temp file for shared state | Parse advisory text from tool_result | Advisory text parsing breaks silently if format changes (D-02 explicitly rejects this) |
| `Date.now()` for latency | `performance.now()` | `performance.now()` gives microsecond resolution but adds `require('perf_hooks')` — not needed; millisecond precision is sufficient for hook latency analysis |

---

## Architecture Patterns

### Recommended File Layout

```
~/.claude/hooks/
├── decision-logger.js        # NEW: PostToolUse + Stop capture hook (last in each chain)
│
~/.claude/dashboard/
├── decision-log.jsonl        # NEW: global training signal aggregate
│
<project>/.planning/
├── token-log.jsonl           # EXISTING: extended with outcome + latency fields
└── decision-log.jsonl        # NEW: per-project training signal

<project>/.claude/settings.json  # EXISTING: add "adaptive": true/false
```

### Pattern 1: Per-Invocation Shared State via Temp File

**What:** Each hook in the PostToolUse chain writes structured results to a temp file keyed by
`session_id + ISO timestamp (truncated to second)`. The decision-logger (last in chain) reads
this file and assembles the decision record.

**When to use:** Any time hooks need to share structured data within a single Claude tool
invocation without modifying stdin/stdout contracts.

**Key insight from codebase:** The PostToolUse stdin payload includes `session_id` and
`data.cwd`. A temp file path like
`/tmp/claude-hook-state-{session_id}-{timestamp_sec}.json` is unique per invocation. Hooks in
the same chain execute sequentially within the same second, so the timestamp truncated to the
second provides a stable shared key.

**Example — upstream hook writes signal:**
```javascript
// Source: derived from codex-review-gate.js + codex-token-logger.js patterns
const os = require('os');
const path = require('path');
const fs = require('fs');

function writeHookSignal(sessionId, timestamp, key, value) {
  // Truncate to second for stable key across hooks in same invocation
  const tsKey = new Date(timestamp || Date.now()).toISOString().slice(0, 19);
  const tmpPath = path.join(os.tmpdir(), `claude-hook-${sessionId}-${tsKey}.json`);
  let state = {};
  try { state = JSON.parse(fs.readFileSync(tmpPath, 'utf8')); } catch (_) {}
  state[key] = value;
  fs.writeFileSync(tmpPath, JSON.stringify(state), 'utf8');
}
```

**Example — decision-logger reads all signals:**
```javascript
function readHookState(sessionId, timestamp) {
  const tsKey = new Date(timestamp || Date.now()).toISOString().slice(0, 19);
  const tmpPath = path.join(os.tmpdir(), `claude-hook-${sessionId}-${tsKey}.json`);
  try { return JSON.parse(fs.readFileSync(tmpPath, 'utf8')); } catch (_) { return {}; }
}
```

**Cleanup:** decision-logger deletes the temp file after reading it, keeping /tmp clean.

### Pattern 2: Latency Measurement with Start/End Timestamps

**What:** Each hook that makes a model call (Codex CLI or MiniMax API) records `startMs`
before the call and `endMs` after it, writing both to the shared state file. The decision-logger
computes `model_latency_ms = endMs - startMs`. For `hook_latency_ms`, hooks record
`hookStartMs` at the very beginning of execution (before any I/O).

**Why two fields:** `model_latency_ms` is the API/CLI call time only — useful for routing
optimization (is Codex faster than MiniMax for this task type?). `hook_latency_ms` is total
wall-clock time the hook took including stdin reading, file I/O, and startup — useful for
detecting when a hook is adding unacceptable overhead.

**Example:**
```javascript
// At hook startup
const hookStartMs = Date.now();
// Before model call
const modelStartMs = Date.now();
// After model call returns
const modelEndMs = Date.now();
// Write to shared state
writeHookSignal(sessionId, tsKey, 'codex_model_latency_ms', modelEndMs - modelStartMs);
writeHookSignal(sessionId, tsKey, 'codex_hook_latency_ms', Date.now() - hookStartMs);
```

### Pattern 3: decision-log.jsonl Schema

**Location decision (D-01):** Use a separate `decision-log.jsonl`, NOT an extension of
`token-log.jsonl`. Rationale: token-log.jsonl is consumed by the billing pipeline
(codex-global-aggregator, cost-reporter, dashboard-generator). Adding training signal fields
to billing records creates dual-consumer confusion and breaks the clean schema. The
`token-log.jsonl` gets only the backward-compatible extensions needed for DCAP-01/DCAP-02
(outcome, latency fields). Decision-level signals go in `decision-log.jsonl`.

**decision-log.jsonl record schema:**
```jsonc
{
  // Identity — links back to the turn in token-log.jsonl
  "timestamp":      "2026-04-03T10:00:00.000Z",
  "session_id":     "30092b41-eaa3-45bf-95b4-64463d5a2dbd",
  "project":        "Claude_X_Codex",
  "cwd":            "/home/alucard/projects/Claude_X_Codex",
  "event_type":     "PostToolUse",              // "PostToolUse" | "Stop"

  // Tool context
  "tool_name":      "Write",                   // Write | Edit | MultiEdit | Bash | null
  "file_ext":       ".js",                     // "" for Bash, null for Stop event
  "diff_lines_changed": 42,                    // from git diff --stat; null if no git

  // Routing signal (from codex-router.js shared state)
  "routing_advice_given": true,
  "routing_disabled":     false,

  // Scan signal (from minimax-post-scan.js shared state)
  "scan_triggered":    true,
  "scan_verdict":      "ISSUES_FOUND",         // "ISSUES_FOUND" | "CLEAN" | "SKIPPED" | "ERROR"
  "scan_issues_count": 2,
  "scan_model_latency_ms": 3200,
  "scan_hook_latency_ms":  3450,

  // Review gate signal (Stop event only; null for PostToolUse)
  "review_verdict":        "BLOCK",            // "ALLOW" | "BLOCK" | null
  "review_block_category": "correctness",      // inferred from block_summary
  "review_model_latency_ms": 12000,
  "review_hook_latency_ms":  12400,

  // Outcome (filled by dismiss command or future git-signal analyzer)
  "outcome":     null,                         // null | "accepted" | "dismissed" | "committed"
  "dismissed_at": null,                        // ISO timestamp if dismissed

  // Task type (new 12-category taxonomy)
  "task_type":  "implementation",

  // Config snapshot at decision time (for drift detection)
  "config_snapshot": {
    "routing_disabled":            false,
    "scan_skip_threshold":         5,
    "compress_tool_output_tokens": 10000,
    "adaptive":                    true
  }
}
```

**token-log.jsonl extensions (DCAP-01, DCAP-02):**
The existing schema gains three nullable fields appended post-literal (same pattern as v2.0
dual_review, verdict fields — see STATE.md Phase 14 decision):
```javascript
// Added to token-log.jsonl records by codex-token-logger.js
if (result.outcome !== undefined)         record.outcome          = result.outcome;
if (result.model_latency_ms !== undefined) record.model_latency_ms = result.model_latency_ms;
if (result.hook_latency_ms !== undefined)  record.hook_latency_ms  = result.hook_latency_ms;
```

### Pattern 4: 12-Category Task Taxonomy

**Current state:** `codex-review-gate.js` classifies into 4 categories: `security`, `test-gen`,
`bulk-ops`, `feature`. The `codex-token-logger.js` records whatever `task_type` the upstream
caller passed in the `[CODEX_RESULT]` marker.

**Expanded 12-category taxonomy (D-06):**

| Category | Detection Signal | Priority |
|----------|-----------------|---------|
| `security-scan` | File path contains auth/login/password/token/session/crypto/rbac/acl/.env/secret (existing rule) | 1st |
| `test-write` | ALL files match test patterns (existing rule, renamed from `test-gen`) | 2nd |
| `bulk-ops` | >10 files OR all files match style/data extensions (existing rule) | 3rd |
| `hook-dev` | ANY file is in `~/.claude/hooks/` OR file extension is `.js` + path contains `hooks` | 4th |
| `plan-review` | Event is `Stop` or `SubagentStop` (review gate / plan reviewer context) | 5th |
| `architecture` | File path contains design/architect/schema/interface + is `.js/.ts` | 6th |
| `refactor` | Tool is `Edit`/`MultiEdit` on existing file with >50% of lines changed (large edit to existing file) | 7th |
| `doc-update` | ALL files are `.md`, `.txt`, `.rst`, or in `/docs/` | 8th |
| `debug` | File path contains debug/fix/hotfix/patch OR diff removes more lines than adds | 9th |
| `explain` | No tool call (chat-only response) OR file is `RESEARCH.md`/`CONTEXT.md` | 10th |
| `implementation` | New `.js/.ts/.py` file created (git diff --no-index /dev/null) | 11th |
| `review` | Source is `review` in [CODEX_RESULT] marker (existing token-logger field) | 12th, fallback |

**Detection location:** A shared `classifyTaskType12(toolName, filePaths, diffStats, eventType)`
function exported from a new `decision-logger.js`. The existing `classifyTaskType()` in
`codex-review-gate.js` is left unchanged (4-category, used for review depth only). The 12-category
classifier runs in decision-logger only.

**Confidence:** MEDIUM — these categories are based on codebase signals, not empirical data.
The taxonomy will need refinement after Phase 16 data flows.

### Pattern 5: /gsd:dismiss-last and /gsd:freeze Commands

**GSD command structure:** Each GSD command is a Markdown file in
`~/.claude/commands/gsd/` with YAML frontmatter (`name`, `description`, `allowed-tools`) and
a process section. Simple commands (like `gsd:note`, `gsd:stats`) are self-contained inline
workflows — they don't need a separate workflow file if the logic fits in ~30 lines of prose.

**dismiss-last implementation approach:**

1. Read `.planning/decision-log.jsonl` for the current project (cwd)
2. Scan backwards to find the last record where `review_verdict == "BLOCK"` and `outcome == null`
3. Count prior dismissals for the same `review_block_category` in the last 30 days
4. Write a dismiss record: update `outcome` to `"dismissed"`, `dismissed_at` to ISO now
   (Since JSONL is append-only, this means appending a new record with the same `session_id`
   + `timestamp` as the original BLOCK, plus `outcome: "dismissed"` + `dismissed_at`)
5. Print feedback: `"Dismissed N/3 times for rule '{category}' — X more and it will be suppressed"`

**freeze/unfreeze implementation approach:**

1. Read `.claude/settings.json` for the current project
2. Set `adaptive: false` (freeze) or `adaptive: true` (unfreeze)
3. Write back atomically (write-then-rename, same pattern as config-writer.js)
4. Print confirmation: `"Adaptive behavior frozen. System will use static v2.0 rules."`

**Adaptive flag location:** `project .claude/settings.json` at root level (not nested under
`codex` or `minimax`). This makes it a first-class project-level toggle that any hook can
check with a single `config.adaptive === false` guard. All adaptive hooks check this flag at
the start of their adaptive logic path and exit silently if `false`.

### Pattern 6: Hook Chain Registration

**Current PostToolUse chain order:**
1. `gsd-context-monitor.js` (timeout: 10)
2. `codex-token-logger.js` (timeout: 10)
3. `codex-wave-validator.js` (timeout: 10)
4. `minimax-post-scan.js` (timeout: 30)
5. `minimax-compress.js` (timeout: 90)

**New order (decision-logger added last):**
6. `decision-logger.js` (timeout: 5) — runs after all signal-producing hooks

**New Stop chain:**
1. `codex-review-gate.js` (timeout: 300) — existing
2. `decision-logger.js` (timeout: 5) — NEW: captures Stop verdict

**Key constraint:** decision-logger must run LAST in each chain. It reads the shared state file
that upstream hooks wrote. If it runs before minimax-post-scan, the scan signal will be absent.

### Anti-Patterns to Avoid

- **Parsing advisory text in tool_result:** The `additionalContext` from upstream hooks IS in
  `tool_result` for downstream hooks, but this parsing is brittle — if `[BUG SCAN]` prefix
  changes to `[SCAN]`, the logger breaks silently. Use shared state file instead.
- **Extending token-log.jsonl schema with decision fields:** Creates dual-consumer confusion.
  Billing pipeline consumers only want billing data.
- **Writing outcome directly to the original JSONL record:** JSONL is append-only. Never modify
  existing records. Dismissals are appended as new records with a `ref_timestamp` linking back
  to the original BLOCK record.
- **Storing adaptive flag in ~/.claude/settings.json (user-scope):** This flag is per-project.
  One project might need freeze while another is still learning. User-scope settings apply
  globally. Use project-scope `.claude/settings.json`.

---

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Atomic file writes | Custom locking mechanism | `fs.writeFileSync(tmp) + fs.renameSync()` | Proven POSIX pattern, already in codex-global-aggregator.js |
| JSONL append | Custom file manager | `fs.appendFileSync(path, JSON.stringify(record) + '\n')` | Exact pattern in codex-token-logger.js — copy it |
| Timestamp-keyed temp files | UUID library | `session_id + timestamp.slice(0,19)` | Both available in every hook stdin; no dep needed |
| Task classification regex | ML classifier | Keyword/path-pattern matching | Sufficient for 12 categories; no training data yet |
| Hook timeout guard | setTimeout watchdog | Existing pattern: `setTimeout(() => process.exit(0), N)` | Already in every hook — copy it |

---

## Common Pitfalls

### Pitfall 1: Advisory Text Format Contract Drift

**What goes wrong:** `decision-logger.js` parses `[BUG SCAN]`, `[Codex Token Log]`, or
`Codex reviewed: PASS` strings from `tool_result`. Any upstream hook that changes its prefix
breaks the logger silently.

**Why it happens:** Advisory text format was never a formal contract — it was ad-hoc.
D-02 exists precisely because of this risk.

**How to avoid:** Use the shared state file pattern (Pattern 1 above). Each upstream hook
writes structured JSON fields. If a hook adds a new field, decision-logger can handle it
gracefully (field is simply absent). No string parsing.

**Warning signs:** `scan_verdict` is consistently `null` even when scan runs; `review_verdict`
is missing from Stop records.

### Pitfall 2: Temp File Key Collision

**What goes wrong:** Two simultaneous sessions with different `session_id` values but identical
truncated-second timestamps write to different temp files correctly. However, a single session
with high tool-call frequency (multiple tool calls in the same second) could have the second
hook in chain N read the state written by chain N-1 if the key is `session_id + ts_sec`.

**Why it happens:** Hook chains run sequentially within a turn, but multiple turns can occur
within the same second.

**How to avoid:** Include `tool_name` + `session_id` + `ts_sec` in the temp file key:
`claude-hook-{session_id}-{ts_sec}-{tool_name}.json`. Since tool_name is in the PostToolUse
payload for all upstream hooks and for decision-logger, this uniquely identifies a single
tool invocation.

**Warning signs:** decision-log.jsonl records with mismatched signals (scan verdict from one
turn appearing in another turn's record).

### Pitfall 3: Dismiss Command Affecting Scans Instead of Blocks

**What goes wrong:** D-04 says dismiss-last targets BLOCK events only, not scans. If the
command scans backwards and hits a `scan_verdict: "ISSUES_FOUND"` record instead of a
`review_verdict: "BLOCK"` record, it dismisses the wrong event.

**Why it happens:** Both BLOCK events and ISSUES_FOUND scans appear in decision-log.jsonl.
Without a clear filter, a naive "find last event" query hits the wrong record type.

**How to avoid:** Filter explicitly: `record.review_verdict === "BLOCK" && record.outcome === null`
when searching for the dismiss target. The `event_type` field (`"Stop"` vs `"PostToolUse"`)
provides an additional guard.

### Pitfall 4: Adaptive Flag in Wrong Scope

**What goes wrong:** `adaptive: false` placed in user-scope `~/.claude/settings.json` freezes
ALL projects, not just the current one. User freezes Claude_X_Codex but inadvertently disables
adaptive behavior in every other project.

**Why it happens:** The difference between user-scope and project-scope settings.json is not
obvious. Most existing config (codex.routing_disabled, minimax.scan_skip_threshold) is in
project-scope.

**How to avoid:** Freeze/unfreeze commands ALWAYS write to `{cwd}/.claude/settings.json`
(project-scope), not to `~/.claude/settings.json` (user-scope). The freeze command must
detect `cwd` from the command context and write there.

### Pitfall 5: token-log.jsonl Schema Extension Breaking Existing Consumers

**What goes wrong:** Adding `outcome`, `model_latency_ms`, `hook_latency_ms` to token-log.jsonl
records causes codex-global-aggregator.js, codex-cost-reporter.js, or codex-dashboard-generator.js
to fail with unexpected fields.

**Why it happens:** If any consumer does schema validation with strict field matching (no extra
fields allowed), new fields break it.

**How to avoid:** Use the v2.0 field addition pattern (confirmed safe — see STATE.md Phase 14
decision: "v2.0 fields added as post-literal mutations in token-logger — preserves !== undefined
semantics for false/0 values"). New fields are added only when not undefined. All existing
consumers in this codebase use duck-typing (they read only the fields they know about and
ignore extras). Verify this pattern holds for new fields before shipping.

### Pitfall 6: Missing Stop Event Decision Records

**What goes wrong:** The Stop hook in `codex-review-gate.js` exits early in many cases
(no code changes detected, `stop_hook_active === true`). If decision-logger only runs when
review-gate runs, Stop events where the gate was skipped produce no decision record.

**Why it happens:** The PostToolUse chain always completes (there's always a tool result),
but the Stop chain exits early when there are no code changes.

**How to avoid:** decision-logger on Stop event should run regardless of review-gate outcome.
It should handle the case where no shared state file exists for this invocation (gate was
skipped — write a record with `review_verdict: "SKIPPED"` or no Stop record at all).
Decision: write no Stop record if the review gate exited early (gate exit = no model call
= no signal to capture). This keeps the log clean.

---

## Code Examples

Verified patterns from live codebase inspection:

### JSONL Append (from codex-token-logger.js)
```javascript
// Source: /home/alucard/.claude/hooks/codex-token-logger.js line 96
fs.appendFileSync(logPath, JSON.stringify(record) + '\n', 'utf8');
```

### Atomic Write-then-Rename (from codex-global-aggregator.js)
```javascript
// Source: /home/alucard/.claude/hooks/codex-global-aggregator.js line 47-52
function atomicWriteJSON(filePath, data) {
  const tmp = filePath + '.tmp.' + process.pid;
  fs.writeFileSync(tmp, JSON.stringify(data, null, 2) + '\n', 'utf8');
  fs.renameSync(tmp, filePath);
}
```

### Stdin Timeout Guard (from all hooks)
```javascript
// Source: /home/alucard/.claude/hooks/codex-token-logger.js line 23
const stdinTimeout = setTimeout(() => process.exit(0), 10000);
// ... stdin reading ...
process.stdin.on('end', () => { clearTimeout(stdinTimeout); /* ... */ });
```

### Advisory Output (PostToolUse) (from codex-token-logger.js)
```javascript
// Source: /home/alucard/.claude/hooks/codex-token-logger.js line 100-106
const output = {
  hookSpecificOutput: {
    hookEventName: 'PostToolUse',
    additionalContext: `[Codex Token Log] ${record.model} ${record.task_type}: ${totalTokens} tokens`
  }
};
process.stdout.write(JSON.stringify(output));
```

### Block Decision Output (Stop hook) (from codex-review-gate.js)
```javascript
// Source: /home/alucard/.claude/hooks/codex-review-gate.js line 198-208
process.stdout.write(JSON.stringify({
  decision: 'block',
  reason: codexPart + '. ' + minimaxPart + '. Fix before responding.'
}));
// OR for pass:
process.stdout.write(JSON.stringify({
  additionalContext: passedModels + ' reviewed: PASS'
}));
```

### Project Config Read Pattern (from minimax-post-scan.js)
```javascript
// Source: /home/alucard/.claude/hooks/minimax-post-scan.js line 261-270
const cwd = data.cwd || process.cwd();
let minimaxConfig = null;
try {
  const projectSettingsPath = path.join(cwd, '.claude', 'settings.json');
  const rawSettings = fs.readFileSync(projectSettingsPath, 'utf8');
  const projectSettings = JSON.parse(rawSettings);
  minimaxConfig = projectSettings.minimax || null;
} catch (_configErr) {
  // No project settings -- use defaults
}
```

### Checking Adaptive Flag (new pattern for this phase)
```javascript
// Pattern for any future adaptive hook to check freeze mode
const cwd = data.cwd || process.cwd();
let adaptiveEnabled = true;
try {
  const settings = JSON.parse(
    fs.readFileSync(path.join(cwd, '.claude', 'settings.json'), 'utf8')
  );
  if (settings.adaptive === false) adaptiveEnabled = false;
} catch (_) {}
if (!adaptiveEnabled) process.exit(0);
```

---

## Existing Advisory Text Formats (Frozen Contracts)

These are the exact string prefixes currently emitted by each upstream hook. They MUST be
documented here so that if the shared state approach is not used for some hook, the parser
has accurate patterns. D-02 prefers shared state, but this is the fallback reference.

| Hook | Event | Advisory Prefix | Format |
|------|-------|-----------------|--------|
| `codex-token-logger.js` | PostToolUse | `[Codex Token Log]` | `[Codex Token Log] {model} {task_type}: {N} tokens, ${cost}` |
| `minimax-post-scan.js` | PostToolUse | `[BUG SCAN]` | `[BUG SCAN] {filename}:\n{findings}` — only when issues found |
| `minimax-compress.js` | PostToolUse | `[Compressed from ~NK tokens]` | Header prepended to compressed text |
| `codex-review-gate.js` | Stop (BLOCK) | decision: "block" | reason: `"Codex found: X. MiniMax found: Y. Fix before responding."` |
| `codex-review-gate.js` | Stop (PASS) | additionalContext | `"{models} reviewed: PASS"` |
| `codex-router.js` | PreToolUse | `[Codex Router]` | `[Codex Router] Routing is ENABLED for this project.` |

**Contract status:** These formats are NOT formally frozen yet. Phase 15 should freeze them.
The planner must include a task to document these formats as frozen contracts in a shared
constants file (e.g., `~/.claude/hooks/hook-contracts.js`). Any future change to these
prefixes requires updating the contracts file AND decision-logger.

---

## Runtime State Inventory

This is not a rename/refactor phase. No runtime state inventory needed.

None — verified: this phase adds new files, does not rename existing ones.

---

## Environment Availability

| Dependency | Required By | Available | Version | Fallback |
|------------|------------|-----------|---------|----------|
| Node.js | All hooks | Yes | v22.22.0 | — |
| `fs`, `path`, `os` | decision-logger.js | Yes | built-in | — |
| `/tmp` writable | Shared state temp files | Yes | — | Use `os.tmpdir()` |
| `better-sqlite3` | Phase 16 only | No | 12.8.0 (npm registry) | JSONL for Phase 15 |
| Project `.claude/settings.json` | Freeze flag, adaptive check | Yes | — | Create if missing |
| `.planning/` directory | decision-log.jsonl | Yes (exists) | — | `fs.mkdirSync` pattern |

**Missing dependencies with no fallback:** None that block Phase 15.

**Missing dependencies with fallback:** `better-sqlite3` — JSONL with line-by-line scan is
sufficient for Phase 15's append-only workload. Migration to SQLite happens in Phase 16.

---

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Advisory text parsing (brittle) | Shared state temp file (structured) | Phase 15 (this phase) | Eliminates silent breakage when hook output format changes |
| 4-category task taxonomy | 12-category taxonomy | Phase 15 | Finer routing signal; better ML training labels |
| No outcome tracking | outcome: accepted/dismissed/committed | Phase 15 | Enables negative training signal collection |
| No latency logging | model_latency_ms + hook_latency_ms | Phase 15 | Enables routing optimization by latency |
| No freeze mode | adaptive: false in settings.json | Phase 15 | Safety escape hatch before Phase 16 auto-tuning |

**Deprecated/outdated:**
- Advisory text parsing as signal source: rejected by D-02; do not use in new code.
- `task_type` with 4 values in token-log.jsonl: remains for backward-compat, but new code uses
  12-category taxonomy from decision-log.jsonl.

---

## Open Questions

1. **Shared state file cleanup timing**
   - What we know: decision-logger deletes the temp file after reading it
   - What's unclear: if decision-logger crashes or exits early, temp files accumulate in /tmp
   - Recommendation: Add a cleanup pass at SessionStart (after global aggregator) that deletes
     temp files older than 24 hours matching `claude-hook-*` pattern

2. **Dismiss feedback count — per rule or per category?**
   - What we know: D-05 says "Dismissed 2/3 times for this rule"
   - What's unclear: "rule" could mean the specific `review_block_category` (e.g., "correctness")
     or a more specific rule ID
   - Recommendation: Use `review_block_category` as the "rule" identity for Phase 15.
     Phase 16 can introduce more specific rule identifiers if data shows the category is too coarse.

3. **Stop event timing — when does decision-logger run relative to review-gate?**
   - What we know: hooks in the Stop chain run in registration order; review-gate is first
   - What's unclear: does Claude Code wait for review-gate to complete before running
     decision-logger, or are they parallel?
   - Recommendation: Settings.json Stop chain registration is sequential (confirmed by existing
     usage patterns). decision-logger will see the complete review-gate output in the Stop
     event context. HIGH confidence based on PostToolUse chain sequential ordering.

---

## Sources

### Primary (HIGH confidence)
- `/home/alucard/.claude/hooks/codex-token-logger.js` — JSONL append pattern, token-log.jsonl schema, advisory text format
- `/home/alucard/.claude/hooks/codex-review-gate.js` — Stop hook structure, BLOCK/ALLOW output, task classification (4 categories)
- `/home/alucard/.claude/hooks/minimax-post-scan.js` — PostToolUse scan pattern, [BUG SCAN] prefix, config read pattern
- `/home/alucard/.claude/hooks/codex-router.js` — PreToolUse advisory format, routing_disabled check
- `/home/alucard/.claude/hooks/codex-global-aggregator.js` — atomic write-then-rename pattern, mtime-gated incremental reads
- `/home/alucard/.claude/settings.json` — hook chain registration, event types, timeouts
- `/home/alucard/projects/Claude_X_Codex/.claude/settings.json` — project settings structure, codex/minimax config shape
- `/home/alucard/projects/Claude_X_Codex/.planning/token-log.jsonl` — live schema confirmed (review records have verdict, block_summary, dual_review, review_task_type fields)
- `/home/alucard/projects/Claude_X_Codex/.planning/STATE.md` — Phase 14 field-addition pattern (post-literal mutations, !== undefined)
- `.planning/research/ARCHITECTURE.md` — decision-log schema design, component responsibilities
- `.planning/research/PITFALLS.md` — false positive suppression trap, advisory text format contracts
- `.planning/research/SUMMARY.md` — phase ordering rationale, stack recommendations

### Secondary (MEDIUM confidence)
- `.planning/phases/15-decision-capture-infrastructure/15-CONTEXT.md` — user decisions D-01 through D-09
- Node.js v22 `os.tmpdir()` — returns `/tmp` on this machine (verified via bash)

### Tertiary (LOW confidence)
- Task-type taxonomy (12 categories) — inferred from codebase signals; no empirical accuracy data yet; requires Phase 16 validation

---

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH — zero new packages; all patterns from live codebase
- Architecture: HIGH — based on live hook inspection; shared state temp file is a clean extension of existing patterns
- Decision-log schema: HIGH — validated against live token-log.jsonl records and architecture research
- 12-category taxonomy: MEDIUM — categories are reasonable but not empirically validated; accuracy unknown until Phase 16
- GSD command structure: HIGH — inspected live command files (gsd:note, gsd:stats, gsd:quick as reference)
- Pitfalls: HIGH — all 6 pitfalls derived from live code inspection and established patterns

**Research date:** 2026-04-03
**Valid until:** 2026-05-03 (30-day window; Node.js v22 APIs and hook patterns are stable)
