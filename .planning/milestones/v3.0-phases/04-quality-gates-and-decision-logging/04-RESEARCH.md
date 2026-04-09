# Phase 4: Quality Gates and Decision Logging - Research

**Researched:** 2026-04-08
**Domain:** Checkpoint systems, feedback loops with hard caps, JSONL decision logging, data integrity validation — all within an existing Node.js Claude Code plugin (Seraphim v3.0)
**Confidence:** HIGH — all findings based on direct code inspection of existing plugin files, canonical research documents, and design spec already in repository

---

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions
None — all Phase 4 decisions are in Claude's Discretion.

### Claude's Discretion
- **D-01:** Checkpoint scope (what constitutes a "task" for checkpoint purposes) — Claude decides based on blueprint task granularity
- **D-02:** Feedback context format (how much prior-phase output gets injected into retry prompts) — Claude decides based on token budget and model context limits
- **D-03:** Cost-gate before loops — Claude decides whether to warn before expensive retry iterations or just run and log
- **D-04:** decisions.jsonl granularity — Claude decides whether to log per-phase-completion, per-executor-call, or both. Must include all required fields: phase, model, profile, tokens_in, tokens_out, cost_usd, latency_ms, outcome, retry_count, loop_count, quality signals
- **D-05:** Data integrity validator implementation approach — Claude decides detection method and reporting format

### Deferred Ideas (OUT OF SCOPE)
None — discussion stayed within phase scope.
</user_constraints>

---

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| QUAL-01 | Between-task checkpoint in Forge: runtime check (tests, lint, imports) + static code review (profile's checkpoint model) on task diff | checkpoint.js dispatches two sub-checks; runtime via Bash; static via existing dispatch.js → executor pattern |
| QUAL-02 | Checkpoint failure triggers retry-with-feedback: Forge re-runs failed task with checkpoint findings appended (max 2 retries per task) | phase-state.js `incrementRetry(projectRoot, phaseId, taskId)` already exists; forge.md Step 6e already stubs for "retry logic is Phase 4" |
| QUAL-03 | Judge->Envision feedback loop: when all approaches get FATAL_FLAW, Envision re-runs with Judge findings (max 2 loops, persisted to disk) | judge.md already emits `loop_required="true"` in PHASE_COMPLETE marker; state.loops.judge_envision counter exists in phase-state.js |
| QUAL-04 | Crucible->Forge feedback loop: verification or adversarial failures trigger targeted Forge fixes with specific instructions (max 2 loops, persisted to disk) | crucible.md already increments state.loops.crucible_forge and checks cap; CRUCIBLE_VERIFY + CRUCIBLE_ADVERSARIAL markers carry issue lists |
| QUAL-05 | When any loop cap is exceeded, pipeline stops and surfaces full findings + suggested manual resolution steps to terminal | Cap logic already partially in judge.md and crucible.md; needs standardized escalation message format |
| QUAL-06 | checkpoint.js branches on project_type: code gets tests+lint, prose gets structure+citation check, science gets methodology+replication check | Design spec defines project_type enum; forge.md already uses projectType from BLUEPRINT marker |
| COST-03 | decisions.jsonl logs every phase execution: phase, model, profile, tokens_in, tokens_out, cost_usd, latency_ms, outcome, retry_count, loop_count | token-logger.js exists in lib/; decisions.jsonl path is .seraphim/decisions.jsonl per project |
| COST-04 | Quality signals in decisions.jsonl: crucible_pass_rate, judge_kill_rate, checkpoint_catch_rate, loop_trigger_reason | Derivable from existing PHASE_COMPLETE marker attributes; append to decisions.jsonl record at phase completion |
| COST-05 | Data integrity validator checks decisions.jsonl for schema violations, missing fields, anomalous values (negative costs, impossible token counts) on session start | Existing decision-logger.js in hooks shows the JSONL append-only pattern; validator reads and validates on load |
</phase_requirements>

---

## Summary

Phase 4 adds the quality gate and logging infrastructure that Phase 3 deliberately left incomplete. The forge.md command already contains a stub at Step 6e: "Do NOT retry in Phase 3 — retry logic is Phase 4 (QUAL-02)". The crucible.md and judge.md commands already implement loop counter increments and cap checks. Phase 4 completes this picture by: (1) building checkpoint.js as the runtime+static review engine, (2) wiring retry-with-feedback into forge.md, (3) implementing the Judge->Envision feedback path in envision.md, (4) adding the Crucible->Forge fix dispatch into forge.md/crucible.md, and (5) building the decisions.jsonl logger and integrity validator.

The implementation is almost entirely additive modifications to existing command files plus two new files (checkpoint.js and a decisions-logger module). All data structures (phase-state.js, markers.js, dispatch.js) exist and are stable. The main design decisions for Claude are: feedback context format (D-02), cost-gate behavior (D-03), and decisions.jsonl granularity (D-04).

**Primary recommendation:** Build checkpoint.js first (QUAL-01 + QUAL-06), then wire retry into forge.md (QUAL-02), then implement feedback loops in judge.md/envision.md/crucible.md (QUAL-03, QUAL-04, QUAL-05), then add decisions.jsonl logger (COST-03, COST-04) and validator (COST-05) last.

---

## Standard Stack

### Core
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| Node.js | v22.22.0 (installed) | All checkpoint and logging scripts | All existing plugin code is Node.js; no new runtime needed |
| `fs` (built-in) | Node stdlib | Synchronous JSONL append, state reads | Used everywhere in existing lib/ files |
| `path` (built-in) | Node stdlib | Plugin-root-relative requires, state paths | Already the project pattern |
| `child_process.execSync` | Node stdlib | Running tests, lint, git diff in checkpoint.js | Same pattern as codex-exec.js spawn calls |

### Existing Plugin Infrastructure (consumed, not built)
| Component | Location | Used By Phase 4 |
|-----------|----------|-----------------|
| `phase-state.js` | `lib/phase-state.js` | `incrementRetry`, `incrementLoop`, `readState`, `writeState` — all loop/retry counters |
| `markers.js` | `lib/markers.js` | Parse PHASE_COMPLETE, APPROACH, CRUCIBLE_VERIFY, CRUCIBLE_ADVERSARIAL markers for loop trigger detection |
| `pricing.js` | `lib/pricing.js` | `computeCostForModel` — feed cost into decisions.jsonl |
| `dispatch.js` | `executors/dispatch.js` | Resolve checkpoint model IDs; dispatch static review call |
| `token-logger.js` | `lib/token-logger.js` | Source of tokens_in/tokens_out for decisions.jsonl |
| `config.js` | `lib/config.js` | Read max_loops, profile for loop cap enforcement |

### No New npm Packages Required
All Phase 4 work uses Node.js stdlib and existing plugin modules. No additional dependencies.

---

## Architecture Patterns

### Recommended Project Structure (additions only)

```
~/.claude/plugins/seraphim/
├── tools/
│   └── checkpoint.js          # NEW: runtime + static review engine (QUAL-01, QUAL-06)
├── lib/
│   └── decisions-logger.js    # NEW: append decisions.jsonl records (COST-03, COST-04)
│   └── decisions-validator.js # NEW: schema + anomaly checker (COST-05)
├── commands/
│   └── forge.md               # MODIFIED: add checkpoint call + retry-with-feedback (QUAL-02)
│   └── envision.md            # MODIFIED: accept judge_findings input for loop path (QUAL-03)
│   └── crucible.md            # MODIFIED: add targeted Forge fix dispatch (QUAL-04)
│   └── judge.md               # MODIFIED: trigger decisions.jsonl write on completion (COST-03)
```

### Pattern 1: Checkpoint as Gatekeeper Between Tasks

**What:** After each Forge task completes, checkpoint.js runs two checks in sequence. Both must pass before advancing to the next task. On failure, the task is retried with findings appended.

**When to use:** Every task in the Forge phase, regardless of project_type (behavior branches inside checkpoint.js).

```javascript
// Source: design spec §Between-task checkpoint + forge.md Step 6 (Phase 3 stub)

// In forge.md — after each task execution (replaces Step 6e stub):
const checkpointResult = await runCheckpoint({
  projectRoot,
  phaseId,
  taskId: task.id,
  taskType: effectiveTaskType,  // from project_type or per-task type attr
  pluginRoot: process.env.CLAUDE_PLUGIN_ROOT
});

if (!checkpointResult.passed) {
  const retryCount = phaseState.incrementRetry(projectRoot, phaseId, task.id);
  const maxRetries = cfg.max_loops || 2;  // D-01: use max_loops as retry cap per design
  if (retryCount > maxRetries) {
    // Escalate — surface to human (QUAL-05)
    appendForgeLog(forgeLogPath, escalationEntry(task, checkpointResult, retryCount));
    throw new Error(`Task ${task.id} exceeded retry cap (${retryCount}/${maxRetries}). Manual resolution required.`);
  }
  // Retry with findings appended (QUAL-02)
  taskPrompt = buildRetryPrompt(taskSpec, checkpointResult.findings);
  // Re-execute task...
}
```

### Pattern 2: checkpoint.js Branch on project_type (QUAL-06)

**What:** checkpoint.js takes `taskType` and runs the appropriate verification strategy.

**When to use:** Called by forge.md after every task.

```javascript
// Source: design spec §Non-Code Project Support + FEATURES.md §Between-Task Checkpoint

function runCheckpoint({ projectRoot, phaseId, taskId, taskType, pluginRoot }) {
  switch (taskType) {
    case 'code':
      return runCodeCheckpoint(projectRoot);     // tests + lint + import check
    case 'research':
    case 'writing':
      return runProseCheckpoint(projectRoot, phaseId, taskId);  // structure + citation
    case 'science':
      return runScienceCheckpoint(projectRoot, phaseId, taskId); // methodology + replication
    default:
      return runCodeCheckpoint(projectRoot);     // safe default
  }
}

function runCodeCheckpoint(projectRoot) {
  // Runtime: npm test / pytest / whatever test runner exists
  const testResult = tryRunTests(projectRoot);
  // Static: dispatch to checkpoint_static model (MiniMax in Performance)
  const staticResult = dispatchStaticReview(projectRoot);
  return {
    passed: testResult.passed && staticResult.passed,
    findings: [...testResult.findings, ...staticResult.findings]
  };
}
```

### Pattern 3: Judge->Envision Feedback Loop (QUAL-03)

**What:** Judge already writes `loop_required="true"` in its PHASE_COMPLETE marker when all approaches receive FATAL_FLAW. Envision needs to accept prior judge findings as additional context.

**Flow:**
1. judge.md sets `loop_required="true"` in PHASE_COMPLETE marker (already done)
2. judge.md reads `state.loops.judge_envision`; if `loopRequired && count < max_loops`, prints "Run /seraphim:envision to retry with findings"
3. envision.md reads judgment.md for `FATAL_FLAW` approach markers and appends them as "## Previous Judge Findings" section in the envision prompt (new logic to add)
4. phase-state.js persists the counter to disk at every increment (already done)

```javascript
// Source: judge.md Step 9 (existing) + envision.md (to be modified)

// In envision.md — detect if this is a loop run:
const judgmentExists = fs.existsSync(judgmentPath);
if (judgmentExists) {
  const judgmentContent = fs.readFileSync(judgmentPath, 'utf8');
  const markers = parseMarkers(judgmentContent);
  const phaseComplete = markers.find(m => m.type === 'PHASE_COMPLETE');
  if (phaseComplete && phaseComplete.loop_required === 'true') {
    // Extract fatal flaw reasons and inject into envision prompt
    const fatalApproaches = markers.filter(m => m.type === 'APPROACH' && m.verdict === 'FATAL_FLAW');
    loopContext = buildLoopContext(fatalApproaches);  // "## Previous Judge Findings\n..."
  }
}
```

### Pattern 4: Crucible->Forge Feedback Loop (QUAL-04)

**What:** When crucible.md reports `verdict="fail"`, Forge re-runs only the affected tasks with targeted fix instructions.

**Flow:**
1. crucible.md already increments `crucible_forge` loop counter and checks cap
2. On fail verdict, crucible.md writes a structured `## Fix Instructions` section in crucible.md
3. forge.md reads crucible.md for `CRUCIBLE_VERIFY` and `CRUCIBLE_ADVERSARIAL` markers — if either is `result="fail"`, forge enters "fix mode": only runs tasks referenced in the issues lists, with fix instructions prepended

```javascript
// Source: crucible.md Step 5 (existing loop check) + Step 9 (output format)

// In forge.md — detect crucible fix mode:
const cruciblePath = path.join(projectRoot, '.seraphim', 'phases', phaseId, 'crucible.md');
if (fs.existsSync(cruciblePath)) {
  const crucibleContent = fs.readFileSync(cruciblePath, 'utf8');
  const markers = parseMarkers(crucibleContent);
  const phaseComplete = markers.find(m => m.type === 'PHASE_COMPLETE' && m.phase === 'crucible');
  if (phaseComplete && phaseComplete.verdict === 'fail') {
    // Build targeted fix task list from issues in crucible.md
    fixMode = true;
    fixInstructions = extractFixInstructions(crucibleContent);
  }
}
```

### Pattern 5: decisions.jsonl Record (COST-03, COST-04)

**What:** Append one record to `.seraphim/decisions.jsonl` at each phase completion.

**Granularity decision (D-04):** Log per-phase-completion (not per-executor-call). This keeps decisions.jsonl focused on phase-level outcomes rather than sub-call noise. The existing token-logger.jsonl already captures per-call data.

```javascript
// Source: design spec §Adaptive Intelligence + FEATURES.md §decisions.jsonl schema

function appendDecision(projectRoot, record) {
  const decisionsPath = path.join(projectRoot, '.seraphim', 'decisions.jsonl');
  const line = JSON.stringify(record) + '\n';
  fs.appendFileSync(decisionsPath, line, 'utf8');
}

// Schema — one record per phase completion:
const record = {
  timestamp: new Date().toISOString(),
  phase: 'forge',           // phase name
  model: forgeModel,        // resolved model ID
  profile: cfg.profile,     // active profile
  tokens_in: usage.input_tokens,
  tokens_out: usage.output_tokens,
  cost_usd: computedCost,
  latency_ms: Date.now() - startMs,
  outcome: 'success',       // success | failure | partial
  retry_count: totalRetries,  // sum of all task retries in this forge run
  loop_count: state.loops.crucible_forge || 0,
  // Quality signals (COST-04):
  quality_signals: {
    crucible_pass_rate: null,         // populated when crucible runs
    judge_kill_rate: null,            // populated when judge runs: fatal / total_approaches
    checkpoint_catch_rate: null,      // populated by forge: checkpoints_failed / checkpoints_run
    loop_trigger_reason: null         // 'all_approaches_fatal' | 'crucible_fail' | 'adversarial_fail' | null
  }
};
```

### Pattern 6: Data Integrity Validator (COST-05)

**What:** On session start (or on-demand), scan decisions.jsonl for invalid records.

**Implementation approach (D-05):** Read the file line-by-line, parse each JSON record, run validation rules. Report violations to stderr or a validation-report.md. Do NOT block execution on validation failure — report and continue (fail-open for review tasks per established convention).

```javascript
// Source: PITFALLS.md §Feedback loop counter persistence + SUMMARY.md §token cost shared formula

const REQUIRED_FIELDS = ['timestamp', 'phase', 'model', 'profile', 'tokens_in', 'tokens_out', 'cost_usd', 'latency_ms', 'outcome'];

function validateDecisions(projectRoot) {
  const decisionsPath = path.join(projectRoot, '.seraphim', 'decisions.jsonl');
  if (!fs.existsSync(decisionsPath)) return { valid: true, violations: [] };

  const lines = fs.readFileSync(decisionsPath, 'utf8').split('\n').filter(l => l.trim());
  const violations = [];

  lines.forEach((line, idx) => {
    let record;
    try { record = JSON.parse(line); }
    catch (e) { violations.push({ line: idx + 1, error: 'JSON parse failure', raw: line.slice(0, 80) }); return; }

    // Missing required fields
    REQUIRED_FIELDS.forEach(field => {
      if (record[field] === undefined) violations.push({ line: idx + 1, field, error: 'missing' });
    });

    // Anomalous values
    if (typeof record.cost_usd === 'number' && record.cost_usd < 0) {
      violations.push({ line: idx + 1, field: 'cost_usd', error: 'negative cost', value: record.cost_usd });
    }
    if (typeof record.tokens_in === 'number' && record.tokens_in < 0) {
      violations.push({ line: idx + 1, field: 'tokens_in', error: 'negative token count', value: record.tokens_in });
    }
    if (typeof record.tokens_out === 'number' && record.tokens_out < 0) {
      violations.push({ line: idx + 1, field: 'tokens_out', error: 'negative token count', value: record.tokens_out });
    }
    // Impossible token counts (>10M per call is nonsensical)
    if (record.tokens_in > 10_000_000 || record.tokens_out > 10_000_000) {
      violations.push({ line: idx + 1, error: 'impossible token count', tokens_in: record.tokens_in, tokens_out: record.tokens_out });
    }
  });

  return { valid: violations.length === 0, violations };
}
```

### Anti-Patterns to Avoid

- **In-memory loop counters:** Never store loop or retry counts in a JS variable without immediately writing to phase-state.js. A crash or session restart would allow the loop to reset and re-run past the cap. `phase-state.js` already writes synchronously on every mutation — use it.
- **Blocking execution on validator failure:** The validator (COST-05) detects anomalies — it does not fix them. Follow the fail-open-for-review-tasks convention: print violations, continue.
- **Sharing retry cap with loop cap:** `max_loops` is the loop cap for Judge->Envision and Crucible->Forge cycles. Forge task retries use the same numeric value by design (all caps are 2 in default config). Do not introduce a separate `max_retries` field — use `cfg.max_loops`.
- **Running full-phase re-execution in Crucible->Forge loop:** Crucible failures are targeted. Only re-run the specific tasks referenced in the CRUCIBLE_VERIFY and CRUCIBLE_ADVERSARIAL issue lists. Re-running all tasks wastes cost and produces redundant forge-log entries.
- **Checkpoint after every file write vs. after every task:** Checkpoint runs between logical blueprint tasks, not after each file write. This is already established in the design spec and FEATURES.md anti-features table.

---

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Loop counter persistence | A Map or object in module scope | `phase-state.js` `incrementLoop` / `incrementRetry` | Already tested, disk-writes on every mutation, crash-safe |
| Marker parsing from phase output files | Custom regex in checkpoint/forge | `markers.js` `parseMarkers` | Already handles escaping, multiline attrs, all marker types |
| Cost computation for checkpoint model | Inline formula | `pricing.js` `computeCostForModel` | Per-provider functions already exist; shared formula causes negative costs (PITFALLS critical item) |
| Model routing for checkpoint model | Hardcoded model names | `dispatch.js` `resolveExecutorId('forge_checkpoint_runtime', cfg)` | Profiles define checkpoint models; dispatch handles overrides and opus_enabled fallback |
| JSONL append | Custom serialization | `fs.appendFileSync` with `JSON.stringify(record) + '\n'` | Existing token-logger uses same pattern; simplest append-only pattern |

**Key insight:** Phase 4 is almost entirely wiring — the infrastructure (phase-state.js, dispatch.js, markers.js, pricing.js) already exists. The work is building checkpoint.js, adding feedback wiring to existing command .md files, and creating the decisions logger.

---

## Common Pitfalls

### Pitfall 1: Loop Counter Lost on Session Restart
**What goes wrong:** If a feedback loop (Judge->Envision or Crucible->Forge) is interrupted mid-run, the in-memory counter resets. On resume, the counter reads 0 from state.json (if not written), allowing the loop to run again past the cap.
**Why it happens:** `incrementLoop` is called but `writeState` is skipped or deferred.
**How to avoid:** `phase-state.js` already calls `writeState` inside every `increment*` function. Always call `phaseState.incrementLoop` (not a manual increment). Never read-modify-write state directly.
**Warning signs:** A Judge->Envision loop runs more than `max_loops` times. Forge retries a task more than twice.

### Pitfall 2: Retry Prompt Without Findings Context
**What goes wrong:** Forge re-runs a failed task with the same prompt as the first attempt. The model produces identical output. The retry fails again.
**Why it happens:** D-02 decision (feedback context format) is not implemented — findings from checkpoint are not injected.
**How to avoid:** The retry prompt MUST include checkpoint findings. Format recommendation (D-02): prepend a `## Checkpoint Findings\n{findings list}` section to the task spec. Keep it to 500 tokens max — include only the specific failure messages, not the full checkpoint output.
**Warning signs:** forge-log.md shows identical "Output summary" for a task and its retry.

### Pitfall 3: decisions.jsonl Negative Cost From Shared Formula
**What goes wrong:** decisions.jsonl records show negative `cost_usd` for Anthropic model calls.
**Why it happens:** `cache_read_input_tokens` are handled as a deduction rather than a discounted charge. From PITFALLS.md: "cache_read tokens are a positive charge at reduced rate (not a credit) — mishandling causes negative cost delta."
**How to avoid:** Always use `pricing.js` `computeCostForModel(modelKey, rawUsage)` — never compute cost inline. This function already has the correct per-provider formulas with the Anthropic cache-read handled correctly.
**Warning signs:** cost_usd < 0 in decisions.jsonl (exactly what COST-05 validator will detect).

### Pitfall 4: checkpoint.js Calling Tests That Don't Exist
**What goes wrong:** For a code project, checkpoint.js runs `npm test` and gets exit code 1 because no test suite exists, or the test command is misconfigured. Every task fails checkpoint. Forge retries everything twice then escalates.
**Why it happens:** Checkpoint assumes a standard test runner is configured; projects may not have one.
**How to avoid:** checkpoint.js must check for test runner configuration before attempting runtime checks. If no test runner is detected (no package.json `test` script, no pytest.ini, no Makefile test target), skip the runtime test step and log "no test runner detected — skipping runtime tests, running static review only." Do not treat absence of tests as a checkpoint failure.
**Warning signs:** All tasks in a forge run fail checkpoint with "no tests found" or "test command not found."

### Pitfall 5: Crucible->Forge Fix Mode Running All Tasks
**What goes wrong:** When crucible.md returns `verdict="fail"`, forge.md re-runs ALL tasks from the blueprint instead of only the tasks referenced in the crucible issues. This re-does work that was already correct, doubles cost, and produces duplicate forge-log entries that confuse Crucible's next pass.
**Why it happens:** Forge doesn't distinguish between "initial run" and "fix run." Fix mode logic is not implemented.
**How to avoid:** Forge must detect crucible.md presence with `verdict="fail"` and build a targeted task list from the issue references. The `## Issues Found` section in crucible.md should contain task IDs. Forge in fix mode only executes tasks whose IDs appear in those lists.
**Warning signs:** forge-log.md shows duplicate entries for the same task IDs in two separate Forge runs.

---

## Code Examples

Verified patterns from existing plugin source:

### Reading loop counter (phase-state.js)
```javascript
// Source: ~/.claude/plugins/seraphim/lib/phase-state.js (verified)
const phaseState = require('${CLAUDE_PLUGIN_ROOT}/lib/phase-state.js');
const state = phaseState.readState(projectRoot, phaseId);
const currentLoops = (state.loops && state.loops['judge_envision']) || 0;
if (currentLoops >= cfg.max_loops) {
  // cap exceeded — escalate
}
const newCount = phaseState.incrementLoop(projectRoot, phaseId, 'judge_envision');
```

### Incrementing retry counter per task
```javascript
// Source: ~/.claude/plugins/seraphim/lib/phase-state.js (verified)
const retryCount = phaseState.incrementRetry(projectRoot, phaseId, taskId);
// taskId is the string from the SERAPHIM:TASK marker id attribute
```

### Parsing existing PHASE_COMPLETE marker for loop_required signal
```javascript
// Source: ~/.claude/plugins/seraphim/commands/judge.md + lib/markers.js (verified)
const { parseMarkers } = require('${CLAUDE_PLUGIN_ROOT}/lib/markers.js');
const judgmentContent = fs.readFileSync(judgmentPath, 'utf8');
const markers = parseMarkers(judgmentContent);
const phaseComplete = markers.find(m => m.type === 'PHASE_COMPLETE');
const loopRequired = phaseComplete && phaseComplete.loop_required === 'true';
```

### Appending to decisions.jsonl
```javascript
// Source: pattern from existing token-logger.js JSONL pattern (verified)
const decisionsPath = path.join(projectRoot, '.seraphim', 'decisions.jsonl');
const record = { timestamp: new Date().toISOString(), phase, model, profile, /* ... */ };
fs.appendFileSync(decisionsPath, JSON.stringify(record) + '\n', 'utf8');
```

### Dispatching static checkpoint review
```javascript
// Source: forge.md §6b dispatch pattern (verified)
const { execSync } = require('child_process');
const promptFile = `/tmp/seraphim-checkpoint-static-${taskId}.md`;
fs.writeFileSync(promptFile, staticReviewPrompt);
const result = execSync(
  `node ${pluginRoot}/executors/dispatch.js --phase forge_checkpoint_static ` +
  `--prompt-file ${promptFile} --project-root ${projectRoot} --output-file /tmp/seraphim-cp-out-${taskId}.md`,
  { timeout: 120_000 }
);
const staticOutput = fs.readFileSync(`/tmp/seraphim-cp-out-${taskId}.md`, 'utf8');
```

### Detecting test runner for code checkpoint
```javascript
// Source: established Node.js pattern for test runner detection
function detectTestRunner(projectRoot) {
  const pkgPath = path.join(projectRoot, 'package.json');
  if (fs.existsSync(pkgPath)) {
    const pkg = JSON.parse(fs.readFileSync(pkgPath, 'utf8'));
    if (pkg.scripts && pkg.scripts.test && pkg.scripts.test !== 'echo "Error: no test specified"') {
      return { command: 'npm test', cwd: projectRoot };
    }
  }
  const pytestIni = path.join(projectRoot, 'pytest.ini');
  const pyprojectToml = path.join(projectRoot, 'pyproject.toml');
  if (fs.existsSync(pytestIni) || fs.existsSync(pyprojectToml)) {
    return { command: 'python3 -m pytest', cwd: projectRoot };
  }
  return null;  // no test runner — skip runtime tests, static only
}
```

---

## Environment Availability

Step 2.6: SKIPPED (no external dependencies — all checkpoint tools are Node.js stdlib + existing executors already verified in Phase 2. No new external services required.)

---

## Open Questions

1. **Cost-gate behavior before loop iterations (D-03)**
   - What we know: Design spec says loops are capped at max_loops (default 2); no explicit pre-loop cost estimate requirement in v3.0 requirements
   - What's unclear: Whether a cost warning is needed before an expensive retry (e.g., Opus-based loop on Performance profile could cost $1-3 per loop)
   - Recommendation: Log loop cost in decisions.jsonl after completion (COST-03 covers this). Do not add a blocking pre-loop cost gate for Phase 4 — it is listed as a v3.0 P3 / v4 feature. Print estimated cost in the loop trigger message but do not block.

2. **Feedback context size limit (D-02)**
   - What we know: Full checkpoint output can be long (especially static review from MiniMax); token budget matters for retry prompts
   - What's unclear: Exact truncation threshold depends on the model running Forge (Codex vs Opus vs Qwen)
   - Recommendation: Cap feedback context at 1000 tokens for the retry prompt injection. Include only failure messages and line numbers — not the full static review prose. If findings exceed 1000 tokens, summarize to the top 5 most critical findings.

3. **forge-log.md format for retry entries (QUAL-02)**
   - What we know: forge.md Step 6c already defines the per-task marker format
   - What's unclear: How to distinguish original attempt vs retry attempt in forge-log.md
   - Recommendation: Add a `retry="N"` attribute to the FORGE_TASK marker for retry entries: `<!-- SERAPHIM:FORGE_TASK id="{id}" status="complete" model="{model}" retry="1" -->`. This makes retry count queryable from the log.

---

## Sources

### Primary (HIGH confidence)
- `~/.claude/plugins/seraphim/lib/phase-state.js` — loop counter API, disk-write-on-increment pattern (direct inspection)
- `~/.claude/plugins/seraphim/lib/markers.js` — SERAPHIM marker parser/emitter (direct inspection)
- `~/.claude/plugins/seraphim/lib/pricing.js` — per-provider cost functions (direct inspection)
- `~/.claude/plugins/seraphim/commands/forge.md` — existing forge flow with Phase 4 stubs (direct inspection)
- `~/.claude/plugins/seraphim/commands/crucible.md` — existing Crucible loop check and cap enforcement (direct inspection)
- `~/.claude/plugins/seraphim/commands/judge.md` — existing judge loop_required signal and counter (direct inspection)
- `.planning/research/FEATURES.md` — checkpoint system table stakes, hard cap rationale (2026-04-04 research)
- `.planning/research/PITFALLS.md` — feedback loop counter persistence pitfall, negative cost formula pitfall (2026-04-04 research)
- `.planning/research/SUMMARY.md` — critical pitfalls summary §5 and §6 (2026-04-04 research)
- `docs/specs/2026-04-04-seraphim-v3-design.md` — feedback loop definitions, checkpoint flow, hard cap rules (authoritative design spec)

### Secondary (MEDIUM confidence)
- `.planning/STATE.md` §Accumulated Context — Phase 3 decisions confirming "Forge does NOT auto-commit (Pitfall 7) — Phase 4 checkpoint owns the commit gate"
- `~/.claude/hooks/decision-logger.js` — JSONL append-only pattern and `writeHookSignal` structure (existing hook, forking pattern)

---

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH — Node.js stdlib + existing plugin modules; no new dependencies
- Architecture: HIGH — existing command files have Phase 4 stubs; data structures are stable
- Pitfalls: HIGH — all five pitfalls derived from verified code inspection and canonical research docs

**Research date:** 2026-04-08
**Valid until:** 2026-06-01 (stable Node.js stdlib; plugin architecture locked; no external API changes affect this phase)
