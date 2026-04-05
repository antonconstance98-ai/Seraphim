# Phase 3: Six Phase Pipeline and Profile Management - Research

**Researched:** 2026-04-04
**Domain:** Claude Code plugin command architecture, multi-model pipeline orchestration, structured output parsing, profile management CLI patterns
**Confidence:** HIGH

---

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions

**Non-Code Project Branching**
- D-01: Forge for non-code projects generates prose/analysis to .md files in the project directory, one file per blueprint task. Commits like code — no difference in the git workflow.
- D-02: Crucible verification for non-code: completeness check — does output cover all blueprint sections? Adversarial: simulated critical peer review — poke holes in arguments, find unsupported claims, logical gaps.
- D-03: `project_type` field in blueprint.md drives branching. Values: `code`, `research`, `writing`, `science`, `mixed`. Forge and Crucible read this field to select behavior.

**Profile Switching**
- D-04: Profile changes are allowed mid-pipeline run. The change takes effect on the NEXT phase — already-completed phases keep their original model assignments. No lock, no error.
- D-05: Phase-state.json records which profile was active when each phase executed, for audit and adaptive intelligence purposes.

**Terminal Output Style**
- D-06: Phase completion output should be "in between" — not a minimal one-liner, not a full verbose dump. Show phase name, model used, key result (e.g., "3 approaches generated", "2 survived", "all tasks passed"), cost. Skip raw token counts and latency unless verbose mode.
- D-07: Headers and banners use the six-winged seraph concept. Seraphim has its own visual identity — not GSD banners. Design phase headers around the six wings theme (each phase = one wing).

### Claude's Discretion
- Command invocation pattern: Claude decides how phase commands call dispatch.js (Bash from .md, direct Node.js, etc.)
- Structured output format: Claude decides between JSON blocks, HTML comment markers, or YAML frontmatter — whatever enables reliable feedback loop parsing
- Banner/header exact ASCII art and formatting — Claude designs within the six-wing theme

### Deferred Ideas (OUT OF SCOPE)
None — discussion stayed within phase scope
</user_constraints>

---

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| PIPE-01 | `/seraphim:discover` runs external + internal research tracks in parallel, writes to `.seraphim/phases/{N}/discovery/external.md` and `internal.md` | Two sub-phases with separate executor calls; parallel_tracks pattern below |
| PIPE-02 | `/seraphim:envision` reads discovery, generates 3-5 approaches with trade-offs, writes `vision.md` | Opus native session via command .md prompt; structured output format research below |
| PIPE-03 | `/seraphim:judge` stress-tests approaches, marks SURVIVES/FATAL_FLAW/CONDITIONAL with machine-readable markers, writes `judgment.md` | HTML comment marker pattern chosen; Gemini dispatch via Bash |
| PIPE-04 | `/seraphim:architect` reads judgment, creates blueprint with task breakdown, writes `blueprint.md` | Opus native session; must include `project_type` field for PIPE-10 |
| PIPE-05 | `/seraphim:forge` executes blueprint task by task with between-task checkpoints, writes forge-log.md | Codex CLI dispatch; checkpoint.js stub (full implementation Phase 4) |
| PIPE-06 | `/seraphim:crucible` verification + adversarial pass, writes `crucible.md` | Dual-pass structure; MiniMax for adversarial; project_type branch per D-02 |
| PIPE-07 | Structured machine-readable markers in phase outputs enabling feedback loop parsing | HTML comment markers chosen — see Structured Output section |
| PIPE-08 | `/seraphim:run {N}` executes all six phases in sequence with auto-advancement | Sequential phase runner in run.md command |
| PIPE-09 | `/seraphim:run {N} --from [phase]` resumes from specific phase | Phase completion state read from phase-state.js; skip completed phases |
| PIPE-10 | Non-code project type support — Forge and Crucible branch on `project_type` | Read from blueprint.md frontmatter; D-01/D-02/D-03 drive behavior |
| PIPE-11 | `/seraphim:new-project` initializes `.seraphim/` directory | Already exists in Phase 1; verify completeness, no new work needed |
| PROF-01 | `/seraphim:set-profile [name]` switches active profile and prints assignment table | config.write() + profiles.json lookup; lists built-in and custom |
| PROF-02 | `/seraphim:show-profile` displays current profile assignments and per-phase cost estimates | config.read() + profiles.json + models.json pricing |
| PROF-03 | `/seraphim:override [phase] [model]` sets per-phase model override | config.read(), mutate overrides, config.write() |
| PROF-04 | `opus_enabled: false` shifts Opus phases to fallback | Already implemented in dispatch.js resolveExecutorId(); command must expose toggle |
| PROF-05 | Balanced and Budget profiles fail gracefully when Qwen unavailable | qwen-exec.js available() returns false gracefully; dispatch fallback chain surfaced visibly |
| PROF-08 | `/seraphim:set-profile` lists both built-in and custom profiles | profiles.json keys + `.seraphim/profiles/` directory scan |
</phase_requirements>

---

## Summary

Phase 3 builds the complete pipeline surface of Seraphim: six slash commands (`/seraphim:discover` through `/seraphim:crucible`), the `/seraphim:run` orchestrator, and five profile management commands. The infrastructure from Phases 1 and 2 is solid — `dispatch.js`, `phase-state.js`, `config.js`, all five executor files, `pricing.js`, and `token-logger.js` all exist and work. Phase 3's job is to wire these pieces together through command `.md` files and agent `.md` files that Opus reads and acts on.

The central design insight for this phase: Claude Code slash commands are Markdown prompt files, not scripts. Opus reads them and decides how to act. For phases that run external models (Judge with Gemini, Forge with Codex), the command instructs Opus to use the Bash tool to call `node executors/dispatch.js` with the phase name and prompt. For phases where Opus is the model (Envision, Architect, Crucible verify), the command instructs Opus directly. This pattern is already proven by the existing `new-project.md` command.

The most complex task in this phase is designing the structured output schemas for each phase (PIPE-07). These schemas must be machine-readable enough for the feedback loop logic in Phase 4 to parse reliably, but human-readable enough for the operator to inspect. HTML comment markers (`<!-- STATUS: -->`) are the recommended format — they survive markdown rendering, don't break the prose flow, and are trivially parseable with a regex.

**Primary recommendation:** Build commands as thin orchestrators that read state, invoke dispatch, and write structured output — avoid putting business logic in .md files; keep it in lib/ JS modules callable by those commands.

---

## Standard Stack

### Core (already installed in Phase 1+2)

| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| Node.js | v22.22.0 | Command orchestration scripts | Project runtime |
| `@google/genai` | 1.48.0 | Gemini API calls | Phase 2 decision (not deprecated @google/generative-ai) |
| `openai` | 6.33.0+ | MiniMax + Perplexity API | Phase 2 decision |
| `fs`, `path`, `child_process` | Node built-in | File I/O, subprocess | No external dep needed |

### Phase 3 Introduces (No New Installs Required)

All logic is in `.md` command files and thin Node.js orchestration scripts. No new npm packages are needed for Phase 3.

**Version verification:** Run `node --version` (v22.22.0 confirmed), `npm list @google/genai` (1.48.0 confirmed), `npm list openai` (6.33.0+ confirmed) at `~/.claude/plugins/seraphim/`.

---

## Architecture Patterns

### Recommended Phase Command Structure

Each phase command `.md` file follows this structure:

```
commands/
├── discover.md          # /seraphim:discover [phase-id]
├── envision.md          # /seraphim:envision [phase-id]
├── judge.md             # /seraphim:judge [phase-id]
├── architect.md         # /seraphim:architect [phase-id]
├── forge.md             # /seraphim:forge [phase-id]
├── crucible.md          # /seraphim:crucible [phase-id]
├── run.md               # /seraphim:run [phase-id] [--from phase]
├── set-profile.md       # /seraphim:set-profile [name]
├── show-profile.md      # /seraphim:show-profile
└── override.md          # /seraphim:override [phase] [model]
```

Agents dir fills out:
```
agents/
├── seraphim-discover.md      # runs external + internal tracks
├── seraphim-envision.md      # Opus-native or dispatch
├── seraphim-judge.md         # dispatches to Gemini via Bash
├── seraphim-architect.md     # Opus-native or dispatch
├── seraphim-forge.md         # dispatches to Codex via Bash; task loop
└── seraphim-crucible.md      # dual-pass verify + adversarial
```

### Pattern 1: Command as Thin Orchestrator

**What:** The `.md` command file instructs Opus to: (1) read config, (2) check prerequisites, (3) call the appropriate agent or dispatch directly, (4) write output file, (5) print summary banner.

**When to use:** Every phase command.

**Why not put logic in .md:** Bash commands in .md are executed by Opus using the Bash tool. Complex branching logic is error-prone in natural language prompts. Keep branching in JS modules; keep .md instructions declarative.

```markdown
---
description: "Run the Judge phase — stress-test all envision approaches"
argument-hint: "[phase-id]"
allowed-tools: ["Read", "Write", "Bash"]
---

## Step 1: Resolve project root and phase ID
[... instructions ...]

## Step 2: Check prerequisites
Read `.seraphim/phases/{phase-id}/vision.md`. If it does not exist, abort with:
"Judge requires Envision output. Run /seraphim:envision {phase-id} first."

## Step 3: Check loop cap
Run: node ${CLAUDE_PLUGIN_ROOT}/lib/phase-state.js getLoopCount {phase-id} judge_envision
[... check against config max_loops ...]

## Step 4: Dispatch to Judge model
Run: node ${CLAUDE_PLUGIN_ROOT}/executors/dispatch.js judge "{prompt}" {project-root}

## Step 5: Write output with structured markers
[... write judgment.md with HTML comment markers ...]

## Step 6: Print wing banner
[... seraph wing N banner + summary line ...]
```

### Pattern 2: Dispatch Call from Command via Bash

**What:** Commands that route to external models (Judge→Gemini, Forge→Codex) use the Bash tool to invoke dispatch.js as a CLI script.

**When to use:** Any phase where the primary model is NOT Opus in the current profile.

**Implementation note:** dispatch.js needs a CLI entry point (when run directly with `node dispatch.js [phase] [prompt-file] [project-root]`) in addition to its existing `module.exports` API.

```bash
# Command calls dispatch with phase slot and path to prompt file
node ${CLAUDE_PLUGIN_ROOT}/executors/dispatch.js \
  --phase judge \
  --prompt-file /tmp/seraphim-judge-prompt.txt \
  --project-root /path/to/project \
  --output-file .seraphim/phases/01-feature/judgment.md
```

**Alternative:** Commands can also `require()` dispatch.js inline if Opus writes a small Node.js runner script to `/tmp/` and executes it. Either approach is valid — the Claude's Discretion clause covers this.

### Pattern 3: Structured Output with HTML Comment Markers

**What:** Each phase writes machine-readable status markers inside its output file using HTML comment syntax.

**When to use:** All six phase outputs. Required for Phase 4 feedback loop parsing.

**Why HTML comments over JSON blocks or YAML frontmatter:**
- JSON blocks inside markdown require a parser to find and extract them; if the model adds prose around the block, the parser breaks
- YAML frontmatter must be at the top of the file; large outputs push it out of context for the parsing model
- HTML comments survive markdown rendering invisibly, can appear anywhere in the file, and are trivially found with a regex: `<!-- SERAPHIM:([^>]+) -->`

```markdown
<!-- SERAPHIM:PHASE_START phase="judge" model="gemini-3-flash" profile="performance" timestamp="2026-04-04T10:00:00Z" -->

# Judgment Report

## Approach 1: Direct State Machine
<!-- SERAPHIM:APPROACH id="approach-1" verdict="SURVIVES" -->
Analysis: ...

## Approach 2: Event-Driven
<!-- SERAPHIM:APPROACH id="approach-2" verdict="FATAL_FLAW" reason="race_condition_on_concurrent_writes" -->
Analysis: ...

## Approach 3: Actor Model
<!-- SERAPHIM:APPROACH id="approach-3" verdict="CONDITIONAL" condition="requires_distributed_lock" -->
Analysis: ...

<!-- SERAPHIM:PHASE_COMPLETE survivors="1" fatal="1" conditional="1" loop_required="false" -->
```

**Regex to parse:**
```javascript
const markers = [];
const re = /<!-- SERAPHIM:(\w+) ([^>]*) -->/g;
let m;
while ((m = re.exec(content)) !== null) {
  const type = m[1];
  const attrs = Object.fromEntries(
    m[2].matchAll(/(\w+)="([^"]+)"/g)
  );
  markers.push({ type, ...attrs });
}
```

### Pattern 4: Six-Wing Banner

**What:** Each phase prints a terminal banner identifying which wing (phase) is running, using the six-wing seraph motif.

**When to use:** Phase start and completion output.

**Design direction (within Claude's Discretion — exact art is Claude's call):**

```
  ═══════════════════════════════════════════════
   ✦  SERAPHIM  ✦  Wing III: The Judge  ✦
  ═══════════════════════════════════════════════
   Model:   Gemini 3 Flash  [performance]
   Phase:   01-feature-name
   Status:  COMPLETE — 2 survived, 1 fatal flaw
   Cost:    $0.04
  ───────────────────────────────────────────────
```

Wings correspond to phases:
- Wing I: The Discoverer
- Wing II: The Envisioner
- Wing III: The Judge
- Wing IV: The Architect
- Wing V: The Forge
- Wing VI: The Crucible

### Pattern 5: Non-Code Project Branching via project_type Field

**What:** The `project_type` field in blueprint.md drives Forge and Crucible behavior per D-03.

**When to use:** Forge and Crucible commands read this field from the blueprint file.

**Implementation:**

The blueprint.md must include a machine-readable frontmatter block at the top:
```markdown
<!-- SERAPHIM:BLUEPRINT project_type="research" phase="01-feature-name" task_count="4" -->
```

Forge command reads this marker first. Branch:
- `code` → instruct Codex to write files and commit
- `research`, `writing` → instruct model to write prose .md files (one per task) and commit
- `science` → instruct model to run analysis scripts, write results .md, commit
- `mixed` → instruct model to handle each task per its declared type

Crucible command reads project_type and branches:
- `code` → goal-backward check against blueprint + adversarial code/security review
- `research`/`writing` → completeness check against blueprint sections + critical peer review
- `science` → methodology check + replication criteria review

### Pattern 6: Phase Prerequisite Chain

**What:** Each phase checks that the prior phase's output file exists before executing.

**When to use:** Every phase command (except Discover, which has no prerequisite).

**Prerequisite chain:**
```
discover  → (no prerequisite)
envision  → requires: discovery/external.md AND discovery/internal.md
judge     → requires: vision.md
architect → requires: judgment.md (AND at least one approach with SURVIVES verdict)
forge     → requires: blueprint.md
crucible  → requires: forge-log.md
```

The prerequisite check reads the output file. For architect, it also parses the SERAPHIM:APPROACH markers to confirm at least one SURVIVES verdict exists.

### Pattern 7: Profile Audit Trail in state.json

**What:** Per D-05, when a phase executes, phase-state.js records the active profile (and model used) in the phase's state.json.

**When to use:** Every phase execution, before writing output file.

**Addition to state.json schema:**
```json
{
  "phase_id": "01-feature-name",
  "created_at": "...",
  "last_updated": "...",
  "completed": true,
  "profile_at_execution": {
    "discover": "performance",
    "envision": "performance",
    "judge": "performance"
  },
  "model_at_execution": {
    "discover_external": "perplexity-sonar",
    "discover_internal": "gemini-3.1-pro",
    "envision": "claude-opus-4-6",
    "judge": "gemini-3-flash"
  },
  "loops": { "judge_envision": 0, "crucible_forge": 0 },
  "retries": {}
}
```

This extends the existing `phase-state.js` schema without breaking the existing API — `writeState` already accepts arbitrary fields.

### Anti-Patterns to Avoid

- **Putting dispatch logic in .md files:** Phase commands must NOT implement the dispatch resolution chain themselves. They call `dispatch.js` or its lib functions. Adding a new model must not require editing any .md file.
- **Writing judgment output before parsing:** The judge command must not write judgment.md until the model output has been validated for the presence of at least one SERAPHIM:APPROACH marker. Otherwise Phase 4 feedback loops get empty files.
- **Silent profile fallback in profile commands:** `/seraphim:set-profile` must not silently accept an invalid profile name. It must validate against profiles.json + `.seraphim/profiles/` before writing.
- **Hardcoded phase names in run.md:** The `/seraphim:run` command must drive phase sequence from an ordered list, not hardcoded `if/else`. This makes the pipeline extensible.
- **Missing profile audit write:** When phase commands skip writing `profile_at_execution` to state.json, the adaptive intelligence layer (Phase 6) cannot correlate outcomes with profiles. Must be in Phase 3.

---

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Profile resolution | Custom profile lookup in each command | `dispatch.js resolveProfile()` | Already handles built-in + custom + validation |
| Executor config resolution | Per-command override logic | `dispatch.js resolveExecutorId()` | Already implements 3-level chain |
| Phase state persistence | In-memory counters in command | `phase-state.js incrementLoop()` | Phase 2 pitfall: counters lost on crash |
| Config read/write | Manual JSON parse in commands | `lib/config.js read()/write()` | Handles defaults, validation, _projectRoot |
| Model cost calculation | Inline cost formulas | `lib/pricing.js` compute functions | Nine different schemas; Phase 2 resolved negative cost bug |
| Token logging | Custom JSONL writes | `hooks/token-logger.js` | Multi-schema normalization already built |
| HTML comment parsing | Custom XML parser | Simple regex (see Pattern 3) | Structured output is intentionally simple; no parser needed |

**Key insight:** Phase 3 is primarily a wiring phase. The hard infrastructure problems were solved in Phases 1 and 2. Commands should be thin shells over existing JS modules.

---

## Common Pitfalls

### Pitfall 1: dispatch.js Has No CLI Entry Point
**What goes wrong:** Commands need to call dispatch from Bash using `node dispatch.js --phase judge ...`. The current dispatch.js only exports functions (`module.exports`). If no CLI entry point (`if (require.main === module)`) exists, the Bash tool call does nothing.
**Why it happens:** dispatch.js was built as a library module for Phase 2. The CLI wrapper is Phase 3 work.
**How to avoid:** Add a CLI entry point block to dispatch.js (or a thin `dispatch-cli.js` wrapper). It must: parse `--phase`, `--prompt-file`, `--project-root`, `--output-file` args; call `resolveExecutorId`; load the correct executor; call `execute()`; write output; exit non-zero on error.
**Warning signs:** `node dispatch.js judge ...` produces no output and exits 0.

### Pitfall 2: Envision Command Confuses Opus-as-Author with Opus-as-Dispatcher
**What goes wrong:** For the Envision phase, Opus IS the execution model (in Performance profile). The command prompt tells Opus to "dispatch to the Envision model." In Performance profile, Opus calls dispatch, dispatch returns `claude-opus-4-6`, which means Opus is supposed to call itself via a subagent. This creates a confusing double-dispatch.
**Why it happens:** The uniform dispatch pattern works cleanly for external models but is awkward when Opus IS the model.
**How to avoid:** Commands for phases where Opus is the model should check `resolveExecutorId('envision', config)` and if the result is `claude-opus-4-6`, instruct Opus to generate the output directly in the current session rather than spawning a subagent. If the result is anything else (e.g., Gemini 3.1 Pro in a non-Performance profile), dispatch via Bash. The conditional is: `if (resolved_model === 'claude-opus-4-6') { do_inline } else { dispatch_via_bash }`.
**Warning signs:** Envision phase spawns a subagent that returns immediately with no output (Opus calling itself).

### Pitfall 3: profile_at_execution Not Written Before Phase Executes
**What goes wrong:** If the command writes `profile_at_execution` to state.json only AFTER the executor returns, a crash mid-execution leaves no profile record. The state file shows the phase never ran, but the output file exists. Phase 4 adaptive intelligence cannot attribute the output to a profile.
**Why it happens:** State update treated as post-processing step.
**How to avoid:** Write `profile_at_execution` and `model_at_execution` to state.json BEFORE calling the executor. Use phase-state.js writeState. The executor may fail, but the audit record must be committed first.
**Warning signs:** Crucible output exists but decisions.jsonl has no corresponding profile record.

### Pitfall 4: run.md Does Not Check Loop State Before Re-Running a Phase
**What goes wrong:** `/seraphim:run {N} --from judge` re-runs Judge without reading the loop counter. If Judge was at `judge_envision: 2` (max loops already hit), run silently continues past the cap, triggering a third Envision cycle.
**Why it happens:** The `--from` resume logic skips phases before the resume point but doesn't inspect loop state.
**How to avoid:** Before starting any phase in `/seraphim:run`, read phase-state.json and check loop counts. If any loop counter is at or above `config.max_loops`, halt and surface a message: "Loop cap reached for judge_envision (2/2). Human resolution required before continuing."
**Warning signs:** Pipeline runs indefinitely across `--from` resumes on a failing phase pair.

### Pitfall 5: set-profile Does Not Record Profile Change in state.json
**What goes wrong:** Per D-04, a mid-run profile change takes effect on the NEXT phase. But if `set-profile` only updates `config.json` without a note in the current phase's state.json, the audit trail (D-05) is broken. The state.json shows phase N ran on profile "performance" but the config.json shows "balanced".
**Why it happens:** set-profile is treated as a config operation, not a pipeline event.
**How to avoid:** When `/seraphim:set-profile` is called and a pipeline is in progress (i.e., at least one phase has a completed state.json), append a `profile_change_events` array to the current phase's state.json: `{ from: "performance", to: "balanced", changed_at: "...", effective_from_next_phase: true }`.
**Warning signs:** decisions.jsonl and config.json show different profiles for consecutive phases with no audit event.

### Pitfall 6: Discover Phase Writes Nothing When a Track Fails
**What goes wrong:** Discover runs two parallel tracks (external and internal). If one fails (e.g., Perplexity API is down), the command writes nothing for that track and the envision phase cannot proceed. The failure is logged to stderr but the operator sees no clear path forward.
**Why it happens:** Parallel track failure handling not specified.
**How to avoid:** If a track fails, write a stub file: `.seraphim/phases/{N}/discovery/external.md` with content `<!-- SERAPHIM:TRACK_FAILED model="perplexity-sonar" error="..." -->`. This satisfies the prerequisite check for Envision while making the failure visible. Envision can proceed with partial discovery; the missing track is surfaced in the wing banner output.
**Warning signs:** Envision fails with "discovery/external.md not found" even when discovery was attempted.

### Pitfall 7: Forge Writes Commits Before Checkpoint Gate
**What goes wrong:** The Forge command instructs Codex to commit after each task. If Codex commits before the checkpoint runs (or if checkpoint is not yet implemented in Phase 3), the commit lands with no review. This makes Phase 4 (quality gates) harder to retrofit.
**Why it happens:** Checkpoint is a Phase 4 deliverable but Forge is Phase 3.
**How to avoid:** In Phase 3, Forge writes to `forge-log.md` after each task but does NOT commit automatically. The commit instruction should be: "write to forge-log.md and display the diff; do not commit until checkpoint passes." Phase 4 adds checkpoint.js and the commit-after-checkpoint logic. This is a deliberate Phase 3 constraint.
**Warning signs:** Forge phase commits with empty test suite and no checkpoint record in forge-log.md.

---

## Code Examples

### CLI Entry Point for dispatch.js

```javascript
// Source: Pattern 2 above + Phase 2 dispatch.js module.exports
// Add to bottom of dispatch.js

if (require.main === module) {
  const args = process.argv.slice(2);
  const phaseIdx = args.indexOf('--phase');
  const promptFileIdx = args.indexOf('--prompt-file');
  const projectRootIdx = args.indexOf('--project-root');
  const outputFileIdx = args.indexOf('--output-file');

  const phase = phaseIdx !== -1 ? args[phaseIdx + 1] : null;
  const promptFile = promptFileIdx !== -1 ? args[promptFileIdx + 1] : null;
  const projectRoot = projectRootIdx !== -1 ? args[projectRootIdx + 1] : process.cwd();
  const outputFile = outputFileIdx !== -1 ? args[outputFileIdx + 1] : null;

  if (!phase || !promptFile) {
    process.stderr.write('[dispatch] --phase and --prompt-file are required\n');
    process.exit(1);
  }

  const config = require('../lib/config').read(projectRoot);
  const modelId = resolveExecutorId(phase, config);
  if (typeof modelId === 'object' && modelId.error) {
    process.stderr.write('[dispatch] ' + modelId.error + '\n');
    process.exit(1);
  }

  const prompt = fs.readFileSync(promptFile, 'utf8');
  const executor = require('./' + modelId + '-exec');
  executor.execute(prompt, { model: modelId }).then(result => {
    if (!result.success) {
      process.stderr.write('[dispatch] Executor failed: ' + result.error + '\n');
      process.exit(1);
    }
    if (outputFile) {
      fs.mkdirSync(path.dirname(outputFile), { recursive: true });
      const tmp = outputFile + '.tmp';
      fs.writeFileSync(tmp, result.output);
      fs.renameSync(tmp, outputFile);
    } else {
      process.stdout.write(result.output);
    }
    process.exit(0);
  }).catch(err => {
    process.stderr.write('[dispatch] Unexpected: ' + err.message + '\n');
    process.exit(1);
  });
}
```

### Structured Output Marker Parsing

```javascript
// Source: Pattern 3 above — parse SERAPHIM HTML comment markers
function parseMarkers(content) {
  const markers = [];
  // Matches: <!-- SERAPHIM:TYPE attr1="val1" attr2="val2" -->
  const re = /<!--\s*SERAPHIM:(\w+)\s+([^>]*?)-->/g;
  let m;
  while ((m = re.exec(content)) !== null) {
    const type = m[1];
    const attrStr = m[2];
    const attrs = {};
    const attrRe = /(\w+)="([^"]*)"/g;
    let a;
    while ((a = attrRe.exec(attrStr)) !== null) {
      attrs[a[1]] = a[2];
    }
    markers.push({ type, ...attrs });
  }
  return markers;
}

// Usage: check Judge output for survivors
const markers = parseMarkers(judgmentContent);
const approaches = markers.filter(m => m.type === 'APPROACH');
const survivors = approaches.filter(m => m.verdict === 'SURVIVES' || m.verdict === 'CONDITIONAL');
const loopRequired = survivors.length === 0;
```

### Profile Table Display for show-profile

```javascript
// Source: Pattern from new-project.md + profiles.json structure
function renderProfileTable(profileName, profileDef, models, config) {
  const lines = [''];
  lines.push('  Profile: ' + profileName + (config.opus_enabled ? '' : ' (opus disabled)'));
  lines.push('');
  lines.push('  Phase                     Model                    Est. Cost');
  lines.push('  ─────────────────────── ─────────────────────── ────────────');

  const phaseLabels = {
    discover_external:        'Discover (external)      ',
    discover_internal:        'Discover (internal)      ',
    envision:                 'Envision                 ',
    judge:                    'Judge                    ',
    architect:                'Architect                ',
    forge:                    'Forge                    ',
    forge_checkpoint_runtime: 'Forge checkpoint (run)   ',
    forge_checkpoint_static:  'Forge checkpoint (static)',
    crucible_verify:          'Crucible (verify)        ',
    crucible_adversarial:     'Crucible (adversarial)   ',
  };

  for (const [slot, label] of Object.entries(phaseLabels)) {
    let modelId = config.overrides && config.overrides[slot]
      ? config.overrides[slot]
      : profileDef.phases[slot];
    const override = config.overrides && config.overrides[slot] ? ' *' : '';
    const modelDef = models[modelId] || {};
    const modelName = (modelDef.name || modelId).padEnd(23);
    // Rough cost per call (assume 2K in + 2K out per phase)
    const inCost = (modelDef.cost_per_mtok?.input || 0) * 0.002;
    const outCost = (modelDef.cost_per_mtok?.output || 0) * 0.002;
    const estCost = (inCost + outCost).toFixed(4);
    lines.push('  ' + label + ' ' + modelName + ' ~$' + estCost + override);
  }

  if (Object.keys(config.overrides || {}).length > 0) {
    lines.push('');
    lines.push('  * = override active');
  }
  lines.push('');
  return lines.join('\n');
}
```

### Phase-State Profile Audit Write

```javascript
// Source: phase-state.js writeState pattern (Phase 2)
// Call BEFORE dispatching to executor

function recordPhaseStart(projectRoot, phaseId, phase, profileName, modelId) {
  const state = phaseState.readState(projectRoot, phaseId);

  if (!state.profile_at_execution) state.profile_at_execution = {};
  if (!state.model_at_execution) state.model_at_execution = {};

  state.profile_at_execution[phase] = profileName;
  state.model_at_execution[phase] = modelId;

  phaseState.writeState(projectRoot, phaseId, state);
}
```

---

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Hooks as pipeline stages | Slash commands + agents as pipeline stages | v3.0 design | Commands are prompt files read by Opus, not scripts |
| Global hooks in settings.json | Plugin hooks via hooks.json | Phase 1 | Hook registration isolated to plugin |
| Single model orchestration | Nine model dispatch with profile presets | v3.0 design | dispatch.js resolves per-phase model assignments |
| @google/generative-ai | @google/genai v1.48.0 | Phase 2 decision | API shape changed; grounding pattern is `{ googleSearch: {} }` |

---

## Open Questions

1. **Discover parallel tracks: truly parallel or sequential?**
   - What we know: PIPE-01 says "in parallel"; the command runs via Opus using Bash tool
   - What's unclear: Opus Bash tool calls are sequential unless the command explicitly spawns background processes. True parallelism requires `&` background jobs or Promise.all from a Node.js script.
   - Recommendation: Run sequentially in Phase 3 (external first, then internal). True parallelism adds complexity without major user-visible benefit at this scale. Document this as a future enhancement.

2. **Forge phase: Codex commit behavior in Phase 3**
   - What we know: PIPE-05 says "writes commits + forge-log.md"; checkpoint is Phase 4
   - What's unclear: Should Forge commit at all in Phase 3, or only write to forge-log.md?
   - Recommendation: Per Pitfall 7, do NOT auto-commit in Phase 3 Forge. Write forge-log.md only. This prevents hard-to-undo commits from an unvalidated checkpoint step.

3. **set-profile PROF-08: listing custom profiles**
   - What we know: Custom profiles live in `.seraphim/profiles/*.json`
   - What's unclear: `.seraphim/` is per-project; if no project exists at CWD, scan would fail
   - Recommendation: set-profile lists built-in profiles always; scans `.seraphim/profiles/` only if `.seraphim/config.json` exists at CWD. If no project, show built-in list only with note: "Run /seraphim:new-project to initialize a project and enable custom profiles."

---

## Environment Availability

| Dependency | Required By | Available | Version | Fallback |
|------------|------------|-----------|---------|----------|
| Node.js | All dispatch/lib calls | Yes | v22.22.0 | — |
| `@google/genai` | Gemini exec (Judge, Discover internal) | Yes | 1.48.0 | — |
| `openai` SDK | MiniMax exec (Crucible adversarial), Perplexity | Yes | 6.33.0+ | — |
| Codex CLI | Forge phase (all profiles) | Yes | 0.118.0+ | — |
| ollama + Qwen 3.5-27B | Balanced/Budget profiles | No (GPU in transit) | — | Fail gracefully per PROF-05 |
| Perplexity API | Performance/Balanced/Moderate discover_external | Yes (configured) | — | Gemini Flash fallback |
| GEMINI_API_KEY | All Gemini phases | Needs verification | — | If missing: gemini-exec.js returns {success:false, error:"no key"} |
| MINIMAX_API_KEY | Judge (non-Performance), Crucible adversarial | Yes (configured) | — | — |
| SearXNG | websearch.sh for Codex/Qwen | Yes (localhost:8888) | — | — |

**Missing dependencies with no fallback:**
- ollama/Qwen: Balanced and Budget profiles blocked until GPU arrives. PROF-05 requires graceful error message.

**Missing dependencies with fallback:**
- GEMINI_API_KEY: If absent, gemini-exec.js already returns structured error (Phase 2). Performance profile Judge phase would fail gracefully and dispatch would surface the error.

---

## Project Constraints (from CLAUDE.md)

| Directive | Applies to Phase 3 |
|-----------|-------------------|
| Plugin path: `~/.claude/plugins/seraphim/` | All new command and agent files go here |
| No runtime dependency on GSD or Superpowers | Commands must not `require()` GSD modules |
| Executors implement `execute/stream/available` | Phase 3 does not add new executors; existing interface respected |
| `dispatch.js` reads `.seraphim/config.json` and routes | Phase commands must not bypass dispatch for external models |
| Phase outputs: `discovery/`, `vision.md`, `judgment.md`, `blueprint.md`, `forge-log.md`, `crucible.md` | These exact filenames are locked — no deviation |
| Temperature 0.01 for MiniMax | Handled in minimax-exec.js (Phase 2); no Phase 3 change needed |
| Token logging to `token-log.jsonl` | dispatch.js CLI entry point must call token-logger after execution |
| Routing decisions to `decisions.jsonl` | dispatch.js CLI must log each dispatch event |
| Hook scripts: Node.js stdin/stdout JSON, 10s timeout guard | Not directly applicable to commands (commands are .md not hooks) |
| Security: Never expose API keys in plaintext | Commands must not echo env vars; errors must not include key values |
| Never bind to 0.0.0.0 | Not applicable to Phase 3 (no servers) |
| Budget: $15/day max | Performance profile ~$3-8/run; within budget |

---

## Sources

### Primary (HIGH confidence)
- Direct code inspection of `~/.claude/plugins/seraphim/` — Phase 1+2 implementation verified
- `docs/specs/2026-04-04-seraphim-v3-design.md` — Approved design spec, all profile tables and architecture diagrams
- `~/.claude/plugins/seraphim/commands/new-project.md` — Established command .md pattern
- `~/.claude/plugins/seraphim/executors/dispatch.js` — Current dispatch implementation (exports only, no CLI entry)
- `~/.claude/plugins/seraphim/lib/phase-state.js` — Full phase state API
- `~/.claude/plugins/seraphim/lib/config.js` — Config read/write/validate API
- `~/.claude/plugins/seraphim/config/profiles.json` — Five profiles, 10 phase slots each, opusFallback defined
- `~/.claude/plugins/seraphim/config/models.json` — Nine models, mechanism, isOpus, pricing

### Secondary (MEDIUM confidence)
- `.planning/research/PITFALLS.md` — Plugin-specific pitfalls (hook double-reg, loop counter persistence, dispatch silent fallback) — cross-verified with Phase 1+2 implementation choices
- `.planning/research/ARCHITECTURE.md` — Component responsibilities and integration points

### Tertiary (LOW confidence)
- None — all critical findings verified against actual codebase

---

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH — all packages already installed and used in Phase 2
- Architecture patterns: HIGH — derived from existing code and approved design spec
- Pitfalls: HIGH — cross-referenced with existing implementation choices and PITFALLS.md
- Structured output format: MEDIUM — HTML comment marker recommendation is reasoned from requirements; exact schema is Claude's Discretion per CONTEXT.md

**Research date:** 2026-04-04
**Valid until:** 2026-05-04 (stable domain — Node.js plugin patterns change slowly)
