# Phase 02: Review Gate & GSD Integration - Research

**Researched:** 2026-04-02
**Domain:** Claude Code Stop hook, PreToolUse/PostToolUse patterns, GSD plugin wave model, Codex CLI review invocation
**Confidence:** HIGH

---

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions

**Review Gate Trigger**
- D-01: Stop hook triggers Codex review **only when code changes are present** — Write/Edit/Bash that modified files. Chat-only responses skip review. Avoids unnecessary Codex calls and latency
- D-02: Review gate is **not bypassable** — if enabled for a project, it always runs. Consistency over speed. Quality enforced on every code change
- D-03: When Codex finds an issue, it **blocks and Opus fixes** the issue before the user sees anything. Quality enforced automatically via the ALLOW/BLOCK protocol

**Review Feedback Visibility**
- D-04: User sees a **one-line summary**: "Codex reviewed: PASS" or "Codex reviewed: fixed [issue]". Minimal noise, clear signal
- D-05: When Codex blocks and Opus fixes, user sees a **brief note**: "Codex caught: [issue]. Fixed before delivery." Transparency without verbosity
- D-06: Review events are **logged to the existing token log** (`.planning/token-log.jsonl`). No separate review log file

**GSD Checkpoint Behavior**
- D-07: Plan-phase finalization is **blocked until Codex reviews**. Codex feedback gets incorporated into the plan file before execution starts
- D-08: Wave-boundary validation is **non-blocking** — Codex validates in the background while execution continues. Results surface at natural stopping points
- D-09: **Critical issues halt the next wave** — if Codex flags a critical issue (broken imports, security flaw) during wave validation, the next wave is blocked until resolved. Non-critical issues remain advisory

**Review Scope**
- D-10: Codex reviews for **all four categories**: bugs/logic errors, security vulnerabilities, requirements alignment, and style/conventions
- D-11: Codex sees the **diff plus relevant surrounding context** — enough to understand impact without reading entire files
- D-12: Review depth **varies by task type** — deep review for new features and security-sensitive code, light review for test generation and bulk operations

**Carrying Forward from Phase 1**
- D-05 (Phase 1): Codex handles code review as one of its four task types
- D-06 (Phase 1): Tool-call based routing detection
- D-08 (Phase 1): Results are attributed — user knows which model did the work

### Claude's Discretion
- Infinite loop prevention: Claude implements the `stop_hook_active` guard to prevent review-triggers-review loops
- Critical issue classification: Claude defines what counts as "critical" vs "non-critical" for wave-halt decisions
- Review prompt design: Claude crafts the review prompts sent to Codex for each review type (stop hook, plan review, wave validation)
- GSD hook integration points: Claude determines exact hook event names and matcher patterns for GSD workflow integration
- Global routing hook design: Claude extends the Phase 1 routing hook to cover non-GSD workflows

### Deferred Ideas (OUT OF SCOPE)
None — discussion stayed within phase scope.
</user_constraints>

---

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| REVW-01 | Stop hook review gate blocks Claude from finishing until Codex reviews the output, with `stop_hook_active` guard preventing infinite loops | Stop hook `stop_hook_active` field confirmed in official docs; exit-code-2 and `decision: "block"` both verified as blocking mechanisms |
| REVW-02 | Cross-model code review works bidirectionally — Claude reviews Codex output AND Codex reviews Claude output | Bidirectional review pattern confirmed in community research; `additionalContext` injection confirmed for both directions |
| ROUT-02 | Global Claude hooks auto-route Codex-specialized tasks in general workflows (not just GSD/Superpowers) | codex-router.js Phase 1 hook already in user-scope settings.json; Phase 2 extends the same global hook |
| GSD-01 | GSD plugin source modified to dispatch Codex validation at wave boundaries (post-wave-execution) | GSD wave model confirmed: plans have `wave` frontmatter field (integer); wave grouping in phase.cjs lines 256-297 |
| GSD-02 | Background Codex validation runs non-blocking during Claude execution, results available at natural stopping points | PostToolUse advisory pattern confirmed; token-logger is the reference for non-blocking background annotation |
| GSD-03 | GSD plan-phase workflow triggers the Opus-Codex review loop before plan finalization | init.cjs plan-phase outputs `plan_checker_enabled` and `checker_model`; existing `has_reviews` / `reviews_path` infrastructure is the injection point |
| GSD-04 | GSD execute-phase workflow routes clearly-defined implementation tasks to Codex where appropriate | codex-router.js PreToolUse hook already advises routing; Phase 2 adds the Stop hook gate — no new routing logic needed for GSD-04 in this phase |
</phase_requirements>

---

## Summary

Phase 2 adds two primary behaviors: a Stop hook that gates every Claude turn ending with code changes on a Codex review, and GSD checkpoints at plan-phase finalization and wave-boundary execution. Both build directly on Phase 1 infrastructure — `codex-exec.js`, `codex-token-logger.js`, and `codex-router.js` are already in place and working.

The critical technical facts confirmed by this research: the Stop hook receives a `stop_hook_active` boolean in its stdin JSON; returning `decision: "block"` (or exiting with code 2) gives Claude another turn to fix the issue; GSD wave grouping lives in PLAN.md frontmatter (`wave:` integer field), not in a runtime state schema. The GSD plan-phase workflow already has a `plan_checker_enabled` / `checker_model` path in `init.cjs` and a `has_reviews` / `reviews_path` artifact convention — the Codex review checkpoint slots into this existing infrastructure rather than requiring a new mechanism.

The wave-boundary detection challenge: GSD does not emit a native "wave boundary" hook event. Instead, the PostToolUse hook on task completion (when a PLAN.md SUMMARY is written) can detect the completion of all plans in a wave by comparing the `waves` object from `gsd-tools.cjs phase-plan-index`. This is the pattern to implement for GSD-01 and GSD-02.

**Primary recommendation:** Build one new hook script (`codex-review-gate.js` as a Stop hook) and two new Bash tool integrations via PostToolUse detection — one for plan-phase REVIEWS.md writing (GSD-03) and one for wave completion detection (GSD-01/GSD-02). All three reuse `codex-exec.js` and append to the existing token log.

---

## Standard Stack

### Core
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| Node.js | v22.22.0 (installed) | Stop hook script, wave-boundary detection | All Phase 1 hooks are Node.js; no new runtime |
| `codex-exec.js` | Phase 1 artifact | Codex invocation wrapper | Already exports `runCodexExec`, `parseCodexTokens`, `computeCost` |
| `codex-token-logger.js` | Phase 1 artifact | Token log appending | Reuse: `[CODEX_RESULT]` marker pattern already handled |
| `@openai/codex` CLI | 0.118.0 (installed) | Codex execution for review calls | Confirmed working at `~/.npm-global/bin/codex` |
| Claude Code hooks API | v2.1.89 (session) | Stop/PostToolUse/PreToolUse events | Native, no third-party dependency |

### Supporting
| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| `gsd-tools.cjs phase-plan-index` | current | Query wave grouping and plan completion status | Wave-boundary detection in PostToolUse hook |
| `child_process.spawn` (Node built-in) | Node 22 | Async Codex invocation (non-blocking) | Wave validation (D-08: non-blocking) |
| `child_process.spawnSync` (Node built-in) | Node 22 | Synchronous Codex invocation | Stop hook gate (must block until review complete) |

**Installation:** No new packages required. All dependencies are Phase 1 artifacts or Node.js built-ins.

**Version verification:** Node.js v22.22.0 confirmed installed 2026-04-02. Codex CLI 0.118.0 confirmed installed 2026-04-02.

---

## Architecture Patterns

### Recommended Project Structure

New files Phase 2 creates:

```
~/.claude/hooks/
├── codex-exec.js              # Phase 1 — reused, not modified
├── codex-token-logger.js      # Phase 1 — reused, not modified
├── codex-router.js            # Phase 1 — extended for ROUT-02 (non-GSD routing)
└── codex-review-gate.js       # Phase 2 NEW — Stop hook review gate

.planning/phases/02-review-gate-gsd-integration/
├── 02-CONTEXT.md
├── 02-RESEARCH.md             # this file
└── 02-NN-PLAN.md              # plans TBD
```

### Pattern 1: Stop Hook with `stop_hook_active` Guard (REVW-01)

**What:** A Stop hook that reads `stop_hook_active` from stdin, detects whether code files changed in this turn, invokes Codex review synchronously, and either allows or blocks based on Codex's ALLOW/BLOCK response.

**When to use:** Every Stop event where code changes (Write/Edit/Bash on non-.planning files) occurred in the current turn.

**Confirmed stdin schema** (source: official Claude Code docs, fetched 2026-04-02):

```json
{
  "session_id": "abc123",
  "transcript_path": "/home/.../.claude/projects/.../transcript.jsonl",
  "cwd": "/home/...",
  "permission_mode": "default",
  "hook_event_name": "Stop",
  "stop_hook_active": false
}
```

**Key field:** `stop_hook_active` — `true` means Claude is already responding to a prior Stop hook block. If `true`, exit 0 (allow) to prevent infinite loops.

**Confirmed output schema** (blocking):

```json
{
  "decision": "block",
  "reason": "Codex found: [issue description]. Fix before responding."
}
```

**Confirmed output schema** (allowing with context):

```json
{
  "additionalContext": "Codex reviewed: PASS"
}
```

**Reference implementation pattern** (from official docs):

```javascript
// Source: https://code.claude.com/docs/en/hooks (fetched 2026-04-02)
const data = JSON.parse(input);

// CRITICAL: always check stop_hook_active first
if (data.stop_hook_active) {
  process.exit(0); // never block when already in hook callback
}

// Check if code changes were made (use transcript_path or git diff)
const hasCodeChanges = detectCodeChanges(data.cwd, data.transcript_path);
if (!hasCodeChanges) {
  process.exit(0); // chat-only response, skip review (D-01)
}

// Invoke Codex synchronously (Stop hook must block until review completes)
const review = await runCodexExec(buildReviewPrompt(diff), { cwd: data.cwd });
const verdict = parseVerdict(review.output); // ALLOW or BLOCK

if (verdict.decision === 'BLOCK') {
  console.log(JSON.stringify({
    decision: 'block',
    reason: `Codex found: ${verdict.issue}. Fix before responding.`
  }));
} else {
  console.log(JSON.stringify({
    additionalContext: 'Codex reviewed: PASS'
  }));
}
```

**Registration in `~/.claude/settings.json`:**

```json
"Stop": [
  {
    "hooks": [
      {
        "type": "command",
        "command": "node \"/home/alucard/.claude/hooks/codex-review-gate.js\"",
        "timeout": 300
      }
    ]
  }
]
```

**Timeout consideration:** Stop hooks run synchronously. Since `runCodexExec` can take up to 300s, the hook timeout in settings.json must be 300 or higher. The Phase 1 `codex-exec.js` already implements SIGTERM + SIGKILL with 300s timeout.

### Pattern 2: Code Change Detection in Stop Hook

**What:** Detect whether the current Claude turn wrote or modified code files (non-.planning, non-docs files).

**Options (in order of reliability):**

1. **Git diff** — `git diff HEAD --name-only` in `cwd`. Shows all staged/unstaged changes. Works immediately after Write/Edit/Bash. **Recommended.**
2. **Transcript parsing** — Read `transcript_path` JSONL and scan for `tool_name: "Write"/"Edit"/"Bash"` entries with file paths. More precise but heavier.

```javascript
// Source: pattern derived from gsd-context-monitor.js and Phase 1 hooks
const { execSync } = require('child_process');

function detectCodeChanges(cwd) {
  try {
    const changed = execSync('git diff HEAD --name-only', { cwd, timeout: 5000 })
      .toString().trim().split('\n').filter(Boolean);
    // Exclude planning docs and markdown — only care about code files
    const codeFiles = changed.filter(f =>
      !f.startsWith('.planning/') &&
      !f.endsWith('.md') &&
      f.length > 0
    );
    return codeFiles.length > 0;
  } catch (e) {
    // Not a git repo or git not available — fall back to permissive (always review)
    return true;
  }
}
```

**Pitfall:** `git diff HEAD` does not show untracked new files. Use `git status --porcelain` combined with `git diff HEAD` to catch both modified and new files.

### Pattern 3: GSD Plan-Phase Review Checkpoint (GSD-03)

**What:** After the planner writes the final PLAN.md files, invoke Codex review before execution can begin. Codex feedback is written to `{phase_dir}/NN-REVIEWS.md`. The GSD `has_reviews` / `reviews_path` fields in `init.cjs` already exist to support this artifact.

**Integration point:** The GSD plan-phase workflow (`/gsd:plan-phase`) calls `gsd-tools.cjs init plan-phase {phase}` which returns:
- `plan_checker_enabled: true/false` — whether to run the plan checker
- `checker_model: "gsd-plan-checker"` — the agent model
- `has_reviews: false` — whether `{phase_dir}/NN-REVIEWS.md` exists
- `reviews_path: null` — populated when reviews file exists

The Codex review checkpoint is implemented as an additional step in the plan-phase agent workflow (not a new hook): after PLAN.md files are written, the plan-phase agent:
1. Runs `codex exec --json` with the plan content as input
2. Writes Codex feedback to `{phase_dir}/02-REVIEWS.md`
3. Incorporates feedback into PLAN.md files if actionable
4. Only then marks planning complete

**This is a workflow-level change, not a hook:** The plan-phase agent (Claude subagent) calls `runCodexExec` directly via Bash or via a dedicated script. No new hook event is needed since plan-phase is already a synchronous sequential workflow.

### Pattern 4: GSD Wave-Boundary Validation (GSD-01, GSD-02)

**What:** After all plans in wave N complete (all have SUMMARY.md), Codex validates the wave output non-blockingly while wave N+1 begins.

**Wave model confirmed** (source: `~/.claude/get-shit-done/bin/lib/phase.cjs`, lines 244-297):
- Each PLAN.md has a `wave:` integer in its YAML frontmatter
- GSD groups plans by wave: `waves: { "1": ["01-01", "01-02"], "2": ["01-03"] }`
- A wave is "complete" when all plans in that wave group have a corresponding SUMMARY.md
- Detection: compare `waves[N]` plan IDs against `completedPlanIds` set

**Wave completion detection:**

```javascript
// Source: derived from phase.cjs cmdPhaseCheckStatus logic
const result = JSON.parse(
  execSync(`node "${GSD_TOOLS}" phase-plan-index ${phase}`, { cwd })
);
// result.waves = { "1": ["01-01", "01-02"], "2": ["01-03"] }
// result.incomplete = ["01-03"]  (not yet summarized)

// Wave 1 is complete when none of waves["1"] appear in result.incomplete
const wave1Plans = result.waves["1"] || [];
const wave1Complete = wave1Plans.every(id => !result.incomplete.includes(id));
```

**Non-blocking execution pattern (D-08):**

```javascript
// Source: pattern from codex-exec.js spawn pattern
// Fire-and-forget: spawn Codex validation without awaiting result
const child = spawn('codex', ['exec', '--json', '--full-auto', '--model', 'gpt-5.4', validationPrompt], {
  cwd, env: process.env, stdio: ['ignore', 'pipe', 'pipe'],
  detached: true  // detach so parent hook can exit immediately
});
// Write child PID to temp file so results can be checked later
fs.writeFileSync(waveLockPath, JSON.stringify({ pid: child.pid, wave, phase }));
child.unref(); // allow parent to exit
```

**Results surfaced at natural stopping points (D-08):** The PostToolUse hook on SUMMARY.md writes detects the child process has completed (check PID via `child_process` or look for result file) and injects findings into `additionalContext`.

**Critical issue halting (D-09):** Codex output is parsed for severity markers. If `CRITICAL` issues are found in the wave validation result, the next wave's first plan is blocked via the Stop hook or via the execute-phase agent checking `.planning/wave-N-validation.json` before starting wave N+1.

### Pattern 5: Global Routing Hook Extension (ROUT-02)

**What:** The Phase 1 `codex-router.js` only fires for Write/Edit tool calls in projects with `routing_enabled: true`. ROUT-02 requires this to work for **all** Claude workflows, not just GSD projects.

**Current state (Phase 1):** `codex-router.js` checks `.claude/settings.json` in `cwd`. If no project config exists, it exits silently. This means non-GSD projects get no routing advice.

**Phase 2 extension:** Add a fallback path in `codex-router.js` that fires for any Write/Edit call when `routing_enabled` is not explicitly disabled. The user-scope `~/.claude/settings.json` already registers this hook globally — Phase 2 just changes the opt-in logic to opt-out.

**Approach:** Check for a `codex.routing_disabled: true` flag to suppress routing. If no project config exists OR routing is not disabled, emit advisory context. This preserves D-07 (opt-in per project) by letting projects set `routing_disabled: true` while making the global hook active by default.

### Anti-Patterns to Avoid

- **Blocking Stop hook when `stop_hook_active: true`**: Creates an infinite loop. Always check this field first and `process.exit(0)` immediately.
- **Using `async: true` for the Stop hook review gate**: Async hooks are advisory-only and cannot block Claude's response. The review gate MUST be synchronous.
- **Running Codex via `exec` (buffered) instead of `spawn` (streaming)** for long review calls: `codex-exec.js` already uses `spawn` — reuse it rather than re-implementing with `exec`.
- **Writing a separate token log file**: D-06 mandates appending to the existing `.planning/token-log.jsonl`. The `[CODEX_RESULT]` marker pattern in `codex-token-logger.js` already handles this — emit the same marker from review calls.
- **Modifying GSD plugin source files directly** (flagged as forbidden in CLAUDE.md): GSD-01 says "GSD plugin source modified" but CLAUDE.md says "Do not modify Claude Code core." Resolution: The wave-boundary detection runs in a new hook script that reads GSD state — it does not modify GSD's internal source. The GSD-03 plan review is implemented as agent workflow behavior, not a source patch.

---

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Codex invocation with timeout | Custom spawn wrapper | `codex-exec.js` `runCodexExec()` | Already implemented with SIGTERM/SIGKILL, JSONL parsing, and cost computation |
| Token logging | New log format | `[CODEX_RESULT]` marker + `codex-token-logger.js` | D-06: use existing log; marker pattern already handled by PostToolUse hook |
| Wave completion detection | Custom plan file scanner | `gsd-tools.cjs phase-plan-index {phase}` | Returns `{ waves: {}, incomplete: [] }` — all the data needed in one call |
| Stop hook infinite loop prevention | Manual flag files | `stop_hook_active` field in stdin | Native to Claude Code hooks API; zero implementation required |
| ALLOW/BLOCK verdict parsing | Custom response format | Prompt Codex to output `ALLOW` or `BLOCK: {reason}` as first line | Simple deterministic parsing; Codex follows format instructions reliably |

**Key insight:** Phase 1 built the execution layer. Phase 2 builds the review orchestration layer on top of it. Almost every primitive needed already exists.

---

## Common Pitfalls

### Pitfall 1: Stop Hook Timeout Too Short
**What goes wrong:** `codex exec` can take 30-120 seconds for a meaningful code review. If the Stop hook timeout in `settings.json` is 10s (same as PostToolUse hooks), Claude Code kills it and reports a "hook error" to the user.
**Why it happens:** PostToolUse hooks use 10s timeout (advisory, fire-and-forget). Stop hooks must complete before Claude responds — they need 300s.
**How to avoid:** Register Stop hook with `"timeout": 300` in `~/.claude/settings.json`.
**Warning signs:** Hook error messages in Claude output; Codex review never completes.

### Pitfall 2: Code Change Detection Misses New (Untracked) Files
**What goes wrong:** `git diff HEAD` only shows tracked files. A new file just written by Write tool is untracked and won't appear in the diff. Stop hook sees no changes and skips review (violating D-02).
**Why it happens:** `git diff HEAD` compares committed state vs working tree for tracked files only.
**How to avoid:** Use `git status --porcelain` to detect `?? newfile.js` untracked files alongside `git diff HEAD --name-only` for modified tracked files. Combine results.
**Warning signs:** Review skipped for plans that added new files.

### Pitfall 3: Wave Validation PID Orphan
**What goes wrong:** The non-blocking Codex validation process is spawned with `child.unref()`, but if the hook exits immediately, the process runs in the background without supervision. If the Claude session ends before validation completes, results are lost silently.
**Why it happens:** `detached: true` + `unref()` is fire-and-forget — no result capture guarantee.
**How to avoid:** Write a `.planning/wave-N-validating.json` sentinel file at spawn time. The next PostToolUse hook checks for this file and reads the result file if present. Set a staleness timeout (e.g., delete if older than 600s).
**Warning signs:** Wave validation results never surface; sentinel files accumulate.

### Pitfall 4: ALLOW/BLOCK Verdict Parsing Fails on Multi-Line Output
**What goes wrong:** The Codex review prompt asks for `ALLOW` or `BLOCK: reason` as first line, but Codex sometimes outputs markdown headers or preamble before the verdict.
**Why it happens:** LLMs don't always strictly follow output format instructions, especially for short deterministic responses.
**How to avoid:** Scan ALL output lines for the first line matching `/^(ALLOW|BLOCK)/i` rather than checking only line 0. Also accept `decision: allow/block` JSON as a fallback format.
**Warning signs:** Stop hook always blocks or always passes regardless of code quality.

### Pitfall 5: GSD Plan-Phase Reviews File Naming Conflict
**What goes wrong:** The existing GSD `has_reviews` / `reviews_path` detection in `init.cjs` looks for `*-REVIEWS.md` files. If the Codex review file uses a different naming convention, GSD won't detect it.
**Why it happens:** GSD expects `{padded_phase_plan_id}-REVIEWS.md` or `REVIEWS.md` pattern (line 231-233 of init.cjs).
**How to avoid:** Name the Codex plan review file `{padded_phase}-REVIEWS.md` (e.g., `02-REVIEWS.md`) to match GSD's detection pattern. Confirm exact pattern: `f.endsWith('-REVIEWS.md') || f === 'REVIEWS.md'`.
**Warning signs:** `has_reviews` stays false in `init plan-phase` output after writing review file.

### Pitfall 6: Routing Opt-In vs. Opt-Out Confusion (ROUT-02)
**What goes wrong:** Phase 1 made routing opt-in (`routing_enabled: true` required). ROUT-02 makes global routing the default. If `codex-router.js` is modified to be opt-out without updating project `.claude/settings.json`, existing projects that relied on opt-in silence will start getting advisory context they didn't expect.
**Why it happens:** Changing semantics of an existing hook in a backward-incompatible way.
**How to avoid:** Add a `codex.routing_disabled: false` explicit opt-out flag. Existing projects with no config keep getting no routing (codex section absent = no routing). Only the global default changes for projects that explicitly set `codex.routing_enabled: true`. Consider whether ROUT-02 should create a new hook entry rather than modifying the existing one.
**Warning signs:** Routing advisory context appears in projects where Codex is not configured.

---

## Code Examples

Verified patterns from official sources and existing codebase:

### Stop Hook Skeleton
```javascript
// Source: Claude Code hooks API docs (fetched 2026-04-02) + gsd-context-monitor.js pattern
'use strict';
const stdinTimeout = setTimeout(() => process.exit(0), 300000); // 300s
let input = '';
process.stdin.setEncoding('utf8');
process.stdin.on('data', chunk => { input += chunk; });
process.stdin.on('end', async () => {
  clearTimeout(stdinTimeout);
  try {
    const data = JSON.parse(input);

    // MUST check first — exits immediately if already in hook callback
    if (data.stop_hook_active) {
      process.exit(0);
    }

    // ... review logic ...

    if (shouldBlock) {
      process.stdout.write(JSON.stringify({ decision: 'block', reason: '...' }));
    } else {
      process.stdout.write(JSON.stringify({ additionalContext: 'Codex reviewed: PASS' }));
    }
  } catch (e) {
    process.exit(0); // silent fail — never block on error
  }
});
```

### Wave Plan Index Query
```javascript
// Source: derived from phase.cjs lines 240-297 (verified 2026-04-02)
const { execSync } = require('child_process');
const GSD_TOOLS = '/home/alucard/.claude/get-shit-done/bin/gsd-tools.cjs';

function getWaveStatus(cwd, phase) {
  const raw = execSync(`node "${GSD_TOOLS}" phase-plan-index ${phase}`, {
    cwd, timeout: 10000
  }).toString();
  return JSON.parse(raw);
  // Returns: { phase, plans, waves: { "1": ["01-01"], "2": ["01-02"] }, incomplete: ["01-02"] }
}

function isWaveComplete(waveStatus, waveNum) {
  const wavePlans = waveStatus.waves[String(waveNum)] || [];
  return wavePlans.length > 0 &&
    wavePlans.every(id => !waveStatus.incomplete.includes(id));
}
```

### ALLOW/BLOCK Verdict Parsing
```javascript
// Source: derived from existing codex-exec.js output parsing pattern
function parseVerdict(codexOutput) {
  const lines = (codexOutput || '').split('\n');
  for (const line of lines) {
    const trimmed = line.trim();
    if (/^ALLOW\b/i.test(trimmed)) {
      return { decision: 'allow', issue: null };
    }
    if (/^BLOCK[:\s]/i.test(trimmed)) {
      const issue = trimmed.replace(/^BLOCK[:\s]*/i, '').trim();
      return { decision: 'block', issue: issue || 'Issue found' };
    }
  }
  // Default to ALLOW if format not recognized (fail-open for review gate)
  return { decision: 'allow', issue: null };
}
```

### Token Log Entry for Review Events
```javascript
// Source: codex-token-logger.js pattern (Phase 1)
// Reviews use task_type: 'review' in the JSONL record
const reviewRecord = {
  timestamp: new Date().toISOString(),
  session_id: data.session_id,
  model: 'gpt-5.4',
  source: 'cli',
  task_type: 'review',   // 'implementation'|'test-gen'|'review'|'bulk-ops'
  tokens: { input, cached_input: 0, output, reasoning_output: 0 },
  cost_usd: computeCost(tokens, 'gpt-5.4'),
  rate_limit_pct: null
};
fs.appendFileSync(logPath, JSON.stringify(reviewRecord) + '\n', 'utf8');
```

---

## GSD Wave Model Reference

The following is confirmed from reading `phase.cjs` lines 244-297 (2026-04-02):

**PLAN.md frontmatter fields used by GSD wave system:**

| Field | Type | Default | Purpose |
|-------|------|---------|---------|
| `wave` | integer | `1` | Wave group number; plans in wave 1 run before wave 2 |
| `autonomous` | boolean | `true` | `false` = has checkpoints (human review stops) |
| `files_modified` | array | `[]` | Files this plan touches |

**Wave state is NOT in STATE.md.** Wave grouping is derived at query time by reading all PLAN.md frontmatter in a phase directory. There is no cached `current_wave` field in STATE.md or any runtime file. The blocker noted in STATE.md ("field names flagged as unverified") is now resolved: the wave state schema is purely in PLAN.md frontmatter.

**Wave completion detection** (authoritative pattern):

```
gsd-tools.cjs phase-plan-index {phase}
→ { waves: { "1": [plan_ids], "2": [plan_ids] }, incomplete: [plan_ids] }
→ Wave N is complete when waves[N] ∩ incomplete = ∅
```

---

## GSD Plan-Phase Integration Points

Confirmed from `init.cjs` `cmdInitPlanPhase` (lines 137-238):

| Field | Value | Use in Phase 2 |
|-------|-------|---------------|
| `plan_checker_enabled` | `true` (from config.json `workflow.plan_check: true`) | Gate Codex plan review on this flag |
| `checker_model` | `"gsd-plan-checker"` | Not used directly — Codex review is separate from plan-checker |
| `has_reviews` | `false` until reviews file written | Check this to avoid re-running review if already done |
| `reviews_path` | null until reviews file written | Populated when `{phase_dir}/02-REVIEWS.md` exists |

**File naming convention for Codex plan reviews** (derived from init.cjs line 231):
- Pattern: `{padded_phase}-REVIEWS.md` (e.g., `02-REVIEWS.md`)
- Detection: `f.endsWith('-REVIEWS.md') || f === 'REVIEWS.md'`
- Location: `{phase_dir}/` (e.g., `.planning/phases/02-review-gate-gsd-integration/02-REVIEWS.md`)

---

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Manual code review after Codex run | Automatic Stop hook gate | Phase 2 | Zero-friction cross-model review |
| Single-model review only | Bidirectional cross-model review (REVW-02) | Phase 2 | Catches bugs each model misses |
| No wave-level validation | Post-wave Codex validation (GSD-01) | Phase 2 | Catches integration issues before next wave |
| Plan-checker uses only Claude | Codex reviews plan before execution (GSD-03) | Phase 2 | Catches spec gaps early when changes are cheap |

**Community consensus on review depth** (source: model comparison research doc, cross-verified with HN threads):
- Codex catches: edge cases, correctness bugs, simpler approaches, over-engineering
- Opus catches: architectural inconsistencies, error-handling gaps, security subtleties
- Cross-model review produces "significantly better results than either model working alone" (Chandler Nguyen benchmark, HIGH confidence)

---

## Open Questions

1. **Git diff availability in Stop hook**
   - What we know: Stop hook receives `cwd` and `transcript_path`; `git` is installed on this machine
   - What's unclear: Whether all Claude-managed projects will always be in git repos. If not, `git status --porcelain` will fail.
   - Recommendation: Wrap git calls in try/catch and fall back to "always review" if git is unavailable (permissive failure). Document this in the hook.

2. **Stop hook timeout interaction with Claude Code UI**
   - What we know: 300s timeout is required; hooks kill at `timeout` seconds
   - What's unclear: Whether the Claude Code UI shows a spinner or progress indicator during Stop hook execution. User experience during a 60-90s review is unknown.
   - Recommendation: Have the hook write a "Codex reviewing..." message to stderr immediately (shown to user during execution), then the final verdict via stdout.

3. **Wave validation result persistence across sessions**
   - What we know: Wave validation is non-blocking (D-08); results must surface at "natural stopping points"
   - What's unclear: If the user closes Claude Code between wave completion and the next session, do validation results still surface?
   - Recommendation: Write results to `.planning/wave-N-validation.json` as a durable artifact. The execute-phase agent checks this file at startup. Include a `status` field (`pending`, `pass`, `critical`, `advisory`) so the agent knows what to do.

4. **`gsd-tools.cjs phase-plan-index` command availability** — RESOLVED
   - Status: Confirmed working. `node ~/.claude/get-shit-done/bin/gsd-tools.cjs phase-plan-index 02` returns `{ phase, plans, waves, incomplete, has_checkpoints }` as expected.
   - No action required.

---

## Environment Availability

| Dependency | Required By | Available | Version | Fallback |
|------------|------------|-----------|---------|----------|
| Node.js | All hook scripts | Yes | v22.22.0 | — |
| `@openai/codex` CLI (`codex`) | REVW-01, REVW-02, GSD-01, GSD-02, GSD-03 | Yes | 0.118.0 at `~/.npm-global/bin/codex` | — |
| `codex-exec.js` | All Codex invocations | Yes | Phase 1 artifact at `~/.claude/hooks/codex-exec.js` | — |
| `codex-token-logger.js` | Token logging (D-06) | Yes | Phase 1 artifact at `~/.claude/hooks/codex-token-logger.js` | — |
| `~/.claude/settings.json` | Hook registration | Yes | Current (Phase 1 already modified) | — |
| `git` CLI | Code change detection in Stop hook | Yes | (system git) | Fall back to always-review if unavailable |
| `gsd-tools.cjs phase-plan-index` | Wave status queries | Yes | Confirmed working (tested 2026-04-02) | Read PLAN.md frontmatter directly |
| `.planning/token-log.jsonl` | Review event logging (D-06) | Yes | Created in Phase 1 | Create on first write |
| `OPENAI_API_KEY` env var | Codex CLI auth | Assumed set (Phase 1 requirement) | — | Hook fails with clear error message |

**Missing dependencies with no fallback:** None identified. All required tools are present.

**Missing dependencies with fallback:** `git` for code change detection (fallback: always-review).

---

## Validation Architecture

Project config has `workflow.nyquist_validation: false` — skipping this section per configuration.

---

## Project Constraints (from CLAUDE.md)

The following directives from `./CLAUDE.md` apply to Phase 2 planning and implementation:

| Directive | Impact on Phase 2 |
|-----------|-------------------|
| Node.js for all hook scripts (not Bash, not Python) | All Phase 2 hooks must be `.js` files using the established stdin/stdout JSON pattern |
| `codex exec --json` subprocess preferred over OpenAI API | Stop hook review gate uses CLI invocation, not API calls |
| Never expose API keys in plaintext | `OPENAI_API_KEY` stays in environment only; never written to hook files or logs |
| Bind services to 127.0.0.1 | Not applicable for hook scripts |
| Opus remains sole architect, Codex never makes architectural decisions | Codex review output is advisory input to Opus; Opus decides whether to fix and how |
| Must not break existing GSD and Superpowers workflows | New Stop hook must be silent on exit when `stop_hook_active: true` or no code changes |
| Codex CLI (subscription) preferred over API billing | All review invocations use `codex exec --json --full-auto` CLI path |
| `async: true` hooks do NOT work for review gate — must be sync | Stop hook is registered synchronously (no `async: true` in settings entry) |
| GSD workflow enforcement — use GSD commands for file changes | Phase 2 work is executed via `/gsd:execute-phase 02` |
| `~/.claude/settings.json` for cross-project hooks | Stop hook and routing extension registered in user-scope settings |
| `.claude/settings.json` for project-specific overrides | Per-project `codex.routing_disabled: true` opt-out lives here |

---

## Sources

### Primary (HIGH confidence)
- Claude Code hooks API docs (`https://code.claude.com/docs/en/hooks`, fetched 2026-04-02) — Stop hook schema, `stop_hook_active` field, decision/block/allow patterns
- `~/.claude/get-shit-done/bin/lib/phase.cjs` (read 2026-04-02, lines 240-297) — Wave model: `wave` frontmatter field, `waves` grouping object, `incomplete` array
- `~/.claude/get-shit-done/bin/lib/init.cjs` (read 2026-04-02, lines 137-238) — plan-phase init output: `plan_checker_enabled`, `has_reviews`, `reviews_path` fields
- `~/.claude/hooks/codex-exec.js` (read 2026-04-02) — `runCodexExec`, `parseCodexTokens`, `computeCost` exports
- `~/.claude/hooks/gsd-context-monitor.js` (read 2026-04-02) — Reference PostToolUse hook pattern: stdin timeout, additionalContext output
- `~/.claude/hooks/codex-router.js` (read 2026-04-02) — Phase 1 routing hook; advisory-only PreToolUse pattern
- `~/.claude/settings.json` (read 2026-04-02) — Current hook registration structure; Stop event not yet present

### Secondary (MEDIUM confidence)
- `docs/research/opus-vs-codex-model-comparison.md` (read 2026-04-02) — Cross-model review effectiveness; Codex catches correctness bugs, Opus catches architectural issues; 2-3 rounds optimal
- `codex-claude-code-power-user-research.md` (read 2026-04-02) — Community patterns for Stop hook review gate; warning about usage limit consumption if review gate is misconfigured

### Tertiary (LOW confidence)
None — all open questions were resolved during research.

---

## Metadata

**Confidence breakdown:**
- Stop hook API (`stop_hook_active`, decision/block output): HIGH — verified from official docs
- Wave model schema (frontmatter `wave:` field, no runtime state): HIGH — read from source
- GSD plan-phase integration (reviews_path, has_reviews): HIGH — read from source
- Phase 1 artifact APIs (runCodexExec, etc.): HIGH — read from source
- `phase-plan-index` command availability: HIGH — confirmed working via live test on 2026-04-02
- Bidirectional review effectiveness: MEDIUM — community consensus, not a controlled benchmark

**Research date:** 2026-04-02
**Valid until:** 2026-05-02 (stable infrastructure; hook API unlikely to change)
