# Architecture Research

**Domain:** Standalone Claude Code plugin — six-phase multi-model orchestration pipeline
**Researched:** 2026-04-04
**Confidence:** HIGH (based on direct inspection of existing codebase, all 18 hooks, settings.json, plugin marketplace examples, and approved design spec)

---

## System Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      CLAUDE CODE SESSION (Opus host)                         │
├─────────────────────────────────────────────────────────────────────────────┤
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐  │
│  │/seraphim:    │  │/seraphim:    │  │/seraphim:    │  │/seraphim:        │  │
│  │  discover    │  │  envision    │  │    judge     │  │  architect       │  │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘  └──────┬───────────┘  │
│         │                 │                 │                  │              │
│  ┌──────┴─────────────────┴─────────────────┴──────────────────┴───────────┐  │
│  │                           dispatch.js                                    │  │
│  │     reads .seraphim/config.json → profile + overrides → executor        │  │
│  └──────────────────────────────────┬────────────────────────────────────────┘  │
├─────────────────────────────────────┼──────────────────────────────────────────┤
│                     EXECUTOR LAYER  │                                           │
│  ┌─────────────┐  ┌─────────────┐   │  ┌─────────────┐  ┌──────────────────┐   │
│  │codex-exec   │  │minimax-exec │   │  │ gemini-exec │  │   qwen-exec      │   │
│  │(fork+adapt) │  │(fork+adapt) │   │  │   (new)     │  │   (new)          │   │
│  └─────────────┘  └─────────────┘   │  └─────────────┘  └──────────────────┘   │
│  ┌─────────────┐                    │                                           │
│  │perplexity-  │                    │                                           │
│  │  exec (new) │                    │                                           │
│  └─────────────┘                    │                                           │
├─────────────────────────────────────┼──────────────────────────────────────────┤
│                      SUPPORT LAYER  │                                           │
│  ┌─────────────┐  ┌─────────────┐   │  ┌─────────────┐  ┌──────────────────┐   │
│  │ pricing.js  │  │  config.js  │   │  │phase-state  │  │  token-logger    │   │
│  │(fork extend)│  │   (new)     │   │  │   (new)     │  │  (fork+adapt)    │   │
│  └─────────────┘  └─────────────┘   │  └─────────────┘  └──────────────────┘   │
├─────────────────────────────────────┼──────────────────────────────────────────┤
│                    EXTERNAL TARGETS │                                           │
│  ┌──────────┐  ┌──────────┐  ┌──────┴──────┐  ┌──────────┐  ┌──────────────┐  │
│  │Codex CLI │  │MiniMax   │  │   Gemini    │  │  Ollama  │  │ Perplexity   │  │
│  │full-auto │  │  API     │  │    API      │  │localhost │  │  MCP / API   │  │
│  └──────────┘  └──────────┘  └─────────────┘  └──────────┘  └──────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Component Responsibilities

| Component | Responsibility | Communicates With |
|-----------|---------------|-------------------|
| `plugin.json` | Plugin manifest — name, version, author. Loaded by Claude Code plugin system. | Claude Code plugin loader |
| `commands/*.md` | Slash command definitions with frontmatter (description, argument-hint, allowed-tools). Markdown prompt files Opus reads and acts on. | Opus session (user invokes) |
| `agents/*.md` | Subagent definitions for phases that Opus spawns. Agents needing non-Claude models call dispatch.js via Bash. | Opus (spawns via Task tool) |
| `dispatch.js` | Reads config, resolves profile + overrides, determines executor, checks availability, calls executor, logs decision. | All executors, lib/config.js, lib/pricing.js |
| `codex-exec.js` | Spawns `codex exec --full-auto --json`, parses JSONL token output. 300s timeout with SIGTERM+SIGKILL. | dispatch.js |
| `minimax-exec.js` | OpenAI SDK with baseURL swap to MiniMax. Temperature 0.01. Exponential backoff retry. | dispatch.js |
| `gemini-exec.js` | `@google/generative-ai` SDK. Search grounding for Discover, thinking mode for Judge, function calling for internal discovery. | dispatch.js |
| `qwen-exec.js` | HTTP to `localhost:11434` ollama. JSON action wrapper executes structured outputs for Forge. 120s timeout. | dispatch.js |
| `perplexity-exec.js` | MCP bridge (preferred) or direct Perplexity API. Returns citations with findings. | dispatch.js |
| `pricing.js` | All nine model pricing tables. `computeCost()` and `computeCostStrict()` functions. | token-logger.js, all executors |
| `config.js` | Reads and writes `.seraphim/config.json`. Provides `resolve(phase)` returning executor ID. | dispatch.js, phase commands |
| `phase-state.js` | Tracks per-project phase completion. Enforces prerequisites. Manages loop counters with hard caps. | commands, dispatch.js |
| `checkpoint.js` | Runs tests + lint (runtime check). Triggers static review via dispatch.js for the checkpoint_static phase. | Forge executor (between tasks) |
| `websearch.sh` | curl wrapper for SearXNG at localhost:8888. Returns JSON. For Codex and Qwen that lack MCP access. | Codex, Qwen tool access |
| `fetchdocs.js` | Context7 HTTP endpoint wrapper. Returns documentation text. | Codex, Qwen tool access |
| `token-logger.js` | Appends JSONL record to `.seraphim/token-log.jsonl` after each model call. Nine model schemas. | Plugin PostToolUse hook |
| `session-start.js` | Loads active profile, reports Seraphim pipeline status at session open. | Plugin SessionStart hook |
| `profiles.json` | Five preset profile definitions (Performance through Frugal). Phase-to-model maps. | dispatch.js, config.js |
| `models.json` | Model registry: access mechanism, pricing keys, capabilities, isOpus flag. | dispatch.js, config.js |

---

## Recommended Plugin Structure

```
~/.claude/plugins/seraphim/
├── plugin.json                    # manifest: name, version, description, author
├── config/
│   ├── profiles.json              # five preset profile definitions
│   └── models.json                # model registry (mechanism, pricing, capabilities)
├── commands/
│   ├── discover.md                # /seraphim:discover [phase-id]
│   ├── envision.md                # /seraphim:envision [phase-id]
│   ├── judge.md                   # /seraphim:judge [phase-id]
│   ├── architect.md               # /seraphim:architect [phase-id]
│   ├── forge.md                   # /seraphim:forge [phase-id]
│   ├── crucible.md                # /seraphim:crucible [phase-id]
│   ├── run.md                     # /seraphim:run [phase-id] [--from phase]
│   ├── set-profile.md             # /seraphim:set-profile [profile-name]
│   ├── show-profile.md            # /seraphim:show-profile
│   ├── override.md                # /seraphim:override [phase] [model]
│   ├── status.md                  # /seraphim:status
│   └── new-project.md             # /seraphim:new-project [name]
├── agents/
│   ├── seraphim-envision.md       # model: opus or profile fallback
│   ├── seraphim-judge.md          # orchestrates external model (Gemini) via Bash
│   ├── seraphim-architect.md      # model: opus or profile fallback
│   ├── seraphim-crucible.md       # model: opus or profile fallback
│   └── seraphim-checkpoint.md     # model: sonnet (Forge runtime check)
├── executors/
│   ├── codex-exec.js              # FORK from ~/.claude/hooks/codex-exec.js
│   ├── minimax-exec.js            # FORK from ~/.claude/hooks/minimax-exec.js
│   ├── gemini-exec.js             # NEW — @google/generative-ai SDK
│   ├── qwen-exec.js               # NEW — ollama HTTP + JSON action wrapper
│   ├── perplexity-exec.js         # NEW — MCP bridge or direct API
│   └── dispatch.js                # NEW — central router
├── tools/
│   ├── websearch.sh               # NEW — SearXNG curl wrapper
│   ├── fetchdocs.js               # NEW — Context7 HTTP wrapper
│   └── checkpoint.js              # NEW — test runner + lint + static review trigger
├── hooks/
│   ├── session-start.js           # NEW (based on existing codex-cost-reporter pattern)
│   └── token-logger.js            # FORK from ~/.claude/hooks/codex-token-logger.js
└── lib/
    ├── pricing.js                 # FORK from ~/.claude/hooks/codex-pricing.js + extend
    ├── config.js                  # NEW — read/write .seraphim/config.json
    └── phase-state.js             # NEW — per-project phase progress tracker
```

### Structure Rationale

- **`plugin.json` at root:** Claude Code plugin loader requires the manifest at the plugin root inside a `.claude-plugin/` subdirectory based on marketplace patterns, or at root for direct installs. Minimal fields suffice. Hooks are NOT declared in plugin.json — they are registered in `~/.claude/settings.json` during plugin setup.
- **`config/`:** Separates static configuration (profiles, model registry) from runtime code. Profiles and models change infrequently and are easier to read/edit as JSON.
- **`commands/`:** One `.md` file per slash command. Frontmatter declares `description`, `argument-hint`, and `allowed-tools`. These are Markdown prompt files — Opus reads them and acts on the instructions. Not scripts.
- **`agents/`:** Subagent `.md` files that Opus spawns via the Task tool. Phases where the primary model is Opus (Envision, Architect) use agents directly. Phases where the primary model is external (Judge with Gemini Flash) use an orchestrating agent that calls dispatch.js via Bash.
- **`executors/`:** All model-specific code isolated here. dispatch.js is the only file that knows which executor to load. Adding a new model = one file + one models.json entry.
- **`tools/`:** Helper scripts for non-MCP-capable models (Codex CLI, Qwen via ollama). These bridge the tool-access gap without requiring those models to have MCP integration.
- **`hooks/`:** Plugin-owned hooks that replace the seven retiring global hooks. Registered in `~/.claude/settings.json` during plugin setup (atomic edit alongside retiring the old hooks).
- **`lib/`:** Shared utilities with no external model dependencies. No network calls. Pure Node.js.

---

## Architectural Patterns

### Pattern 1: Unified Executor Interface

**What:** Every model executor exports exactly three functions: `execute(prompt, opts)`, `stream(prompt, opts)`, and `available()`. dispatch.js never calls executor-specific methods.

**When to use:** Every time a phase command needs to invoke a model.

**Trade-offs:** Forces adapter work for models with incompatible output formats (Codex JSONL vs. Gemini streaming). Worth it — adding a new model is one file with no changes to dispatch.js or any command.

```javascript
// Standard executor contract — every executor must implement exactly this
module.exports = {
  // Returns: { success, output, tokens, cost, error }
  // tokens: { input_tokens, cached_input_tokens, output_tokens }
  // cost: USD float (0 for Codex CLI and Qwen local)
  execute(prompt, opts) {},

  // Returns ReadableStream where supported; falls back to execute() otherwise
  stream(prompt, opts) {},

  // Returns boolean — is this model reachable right now?
  // codex-exec:      checks `which codex` and OPENAI_API_KEY presence
  // minimax-exec:    checks MINIMAX_API_KEY presence
  // gemini-exec:     checks GEMINI_API_KEY presence
  // qwen-exec:       HTTP GET localhost:11434/api/tags (ollama running check)
  // perplexity-exec: checks MCP availability or PERPLEXITY_API_KEY
  available() {}
};
```

### Pattern 2: Config Resolution Chain

**What:** dispatch.js resolves model assignments through a three-level priority chain: per-phase override > opus_enabled flag > profile preset. Returns an executor ID string.

**When to use:** Every dispatch.js invocation begins here.

**Trade-offs:** Slightly more complex than a flat config. Enables "presets not locks" — the core design invariant.

```javascript
// dispatch.js resolution (pseudo-code)
function resolveExecutorId(phase, config) {
  const profiles = require('../config/profiles.json');
  const models   = require('../config/models.json');

  // Level 1: per-phase override in .seraphim/config.json
  if (config.overrides && config.overrides[phase]) {
    return config.overrides[phase];
  }

  // Level 2: profile preset
  const profileDef = profiles[config.profile];
  let modelId = profileDef.phases[phase];

  // Level 3: opus_enabled:false → use profile's opus fallback
  if (!config.opus_enabled && models[modelId] && models[modelId].isOpus) {
    modelId = profileDef.opusFallback;
  }

  return modelId;
}
```

### Pattern 3: Per-Profile Fallback Chain (owned by dispatch.js)

**What:** When an executor's `available()` returns false, dispatch.js consults the profile's `fallbackChain` array and tries executors in order.

**When to use:** Any time a model is unreachable (API key missing, ollama not running, Codex rate-limited).

**Trade-offs:** Fallback logic is now in dispatch.js, not inside individual executors. The existing `runWithFallback()` in minimax-exec.js is NOT forked into the plugin — its logic moves to dispatch.js.

```javascript
// dispatch.js fallback chain
async function dispatchWithFallback(phase, prompt, opts, config) {
  const profile = profiles[config.profile];
  const chain = profile.fallbackChain[phase] || [resolveExecutorId(phase, config)];

  for (const executorId of chain) {
    const executor = require(`../executors/${executorId}`);
    if (!executor.available()) continue;
    const result = await executor.execute(prompt, opts);
    if (result.success) return { ...result, executorUsed: executorId };
  }

  return { success: false, error: 'All executors in fallback chain unavailable' };
}
```

### Pattern 4: Feedback Loop with Hard Cap

**What:** Judge can send Envision back for re-run. Crucible can send Forge back. Both loops have a counter in phase-state.js, checked and incremented before re-invoking.

**When to use:** Phase outputs that must pass a quality gate before proceeding.

**Trade-offs:** Without hard caps, a stuck loop burns money. The cap must be checked before spawning, not after.

```javascript
// lib/phase-state.js
function canLoop(phaseStateFile, loopType, maxLoops) {
  const max = maxLoops || 2;
  const state = readState(phaseStateFile);
  const count = (state.loops && state.loops[loopType]) || 0;
  if (count >= max) return false;
  writeState(phaseStateFile, {
    ...state,
    loops: { ...(state.loops || {}), [loopType]: count + 1 }
  });
  return true;
}
```

### Pattern 5: Decision Logging for Adaptive Intelligence

**What:** Every dispatch.js call appends a JSONL record to `.seraphim/decisions.jsonl` after the executor resolves. Captures model, phase, profile, tokens, cost, latency, outcome, and quality signals.

**When to use:** After every executor.execute() call, success or failure.

**Trade-offs:** Small I/O overhead per call. Essential — without accumulated decisions, recommendations are guesswork rather than evidence.

```javascript
// decisions.jsonl record shape
{
  "timestamp": "2026-04-04T10:00:00.000Z",
  "phase": "judge",
  "model": "gemini-3-flash",
  "profile": "performance",
  "tokens": { "input": 4200, "output": 800 },
  "cost_usd": 0.00374,
  "latency_ms": 3200,
  "outcome": "success",         // "success" | "failure" | "retry"
  "retry_count": 0,
  "loop_count": 0,              // feedback loops triggered before this call
  "quality_signal": {
    "approaches_surviving": 3,  // Judge: how many approaches survived
    "crucible_issues_found": 0  // Crucible: adversarial issues found
  }
}
```

---

## Data Flow

### Full Six-Phase Pipeline (Performance profile)

```
User: /seraphim:run 01-my-feature
  |
  +--> commands/run.md (Opus reads, orchestrates sequence)
        |
        +--> DISCOVER
        |     +--> dispatch.js("discover_external") → perplexity-exec.js → Perplexity MCP
        |     +--> dispatch.js("discover_internal") → gemini-exec.js → Gemini 3.1 Pro API
        |     +--> Writes: .seraphim/phases/01-my-feature/discovery/{external,internal}.md
        |
        +--> ENVISION
        |     +--> seraphim-envision agent (native Opus session)
        |     +--> Reads:  .seraphim/phases/01-my-feature/discovery/*.md
        |     +--> Writes: .seraphim/phases/01-my-feature/vision.md
        |
        +--> JUDGE [feedback loop: max 2 → Envision]
        |     +--> dispatch.js("judge") → gemini-exec.js → Gemini 3 Flash API
        |     +--> Reads:  .seraphim/phases/01-my-feature/vision.md
        |     +--> Writes: .seraphim/phases/01-my-feature/judgment.md
        |     +--> All FATAL? → phase-state.canLoop("judge-envision")? → re-run ENVISION
        |
        +--> ARCHITECT
        |     +--> seraphim-architect agent (native Opus session)
        |     +--> Reads:  .seraphim/phases/01-my-feature/judgment.md (top survivor)
        |     +--> Writes: .seraphim/phases/01-my-feature/blueprint.md
        |
        +--> FORGE [per-task checkpoint; max 2 retries per task]
        |     +--> dispatch.js("forge") → codex-exec.js → codex exec --full-auto
        |     +--> Reads:  .seraphim/phases/01-my-feature/blueprint.md (task list)
        |     +--> After each task:
        |     |     +--> checkpoint.js runtime check (tests + lint)
        |     |     +--> dispatch.js("forge_checkpoint_static") → minimax-exec.js
        |     |     +--> Both pass? → next task
        |     |     +--> Either fails? → phase-state retry counter → retry with feedback
        |     +--> Writes: forge-log.md + git commits
        |
        +--> CRUCIBLE [outer loop: max 2 → Forge]
              +--> dispatch.js("crucible_verify") → Opus session (native)
              +--> dispatch.js("crucible_adversarial") → minimax-exec.js
              +--> Reads:  blueprint.md + forge output
              +--> Writes: .seraphim/phases/01-my-feature/crucible.md
              +--> FAIL? → phase-state.canLoop("crucible-forge")? → re-run FORGE
```

### dispatch.js Internal Flow

```
dispatch.js(phase, prompt, opts)
  |
  +--> lib/config.js.read(cwd)       → { profile, opus_enabled, overrides }
  +--> resolveExecutorId(phase, cfg) → executorId (e.g., "gemini-exec")
  +--> executor = require(`../executors/${executorId}`)
  +--> executor.available()?
  |     No → try profile.fallbackChain[phase] in order
  +--> result = await executor.execute(prompt, opts)
  +--> appendDecision(decisions.jsonl, { phase, executorId, result, latency })
  +--> return result
```

### Hook Signal Flow After Migration

```
Claude writes/edits a file (PostToolUse fires):
  |
  +--> ~/.claude/settings.json PostToolUse chain (post-migration):
        +--> gsd-context-monitor.js     (KEPT — global hook, unchanged)
        +--> seraphim/hooks/token-logger.js  (PLUGIN — replaces codex-token-logger.js)
        +--> decision-logger.js         (KEPT — global hook, unchanged)
        [REMOVED: codex-wave-validator.js]
        [REMOVED: minimax-post-scan.js]
        [REMOVED: minimax-compress.js]

Claude session ends (Stop fires):
  |
  +--> ~/.claude/settings.json Stop chain (post-migration):
        [REMOVED: codex-review-gate.js]
        [REMOVED: codex-plan-reviewer.js]
        [REMOVED: codex-multi-round-reviewer.js]
        [REMOVED: codex-superpowers-plan-reviewer.js]
        (review now happens inside Crucible phase, not on every Stop)
```

---

## Integration Points

### New Plugin vs. Existing Hooks

| Existing Hook | Chain | Action | Seraphim Replacement |
|--------------|-------|--------|---------------------|
| `codex-review-gate.js` | Stop | RETIRE | Crucible adversarial pass |
| `codex-plan-reviewer.js` | Stop/SubagentStop | RETIRE | Judge phase |
| `codex-multi-round-reviewer.js` | Stop/SubagentStop | RETIRE | Judge phase (multi-loop) |
| `codex-superpowers-plan-reviewer.js` | Stop/SubagentStop | RETIRE | Judge phase |
| `minimax-post-scan.js` | PostToolUse | RETIRE | Crucible adversarial pass |
| `minimax-compress.js` | PostToolUse | RETIRE | Each executor manages its own context |
| `codex-router.js` | PreToolUse | RETIRE | dispatch.js |
| `codex-wave-validator.js` | PostToolUse | RETIRE | Forge checkpoint.js |
| `codex-token-logger.js` | PostToolUse | REPLACE with plugin fork | `hooks/token-logger.js` (nine models) |
| `codex-cost-reporter.js` | SessionStart | KEEP (global) | Seraphim adds its own session-start.js |
| `codex-global-aggregator.js` | SessionStart | KEEP + extend | New Seraphim metrics panel added |
| `decision-logger.js` | PostToolUse+Stop | KEEP (global) | Separate from decisions.jsonl |
| `gsd-context-monitor.js` | PostToolUse | KEEP | No change |
| `gsd-check-update.js` | SessionStart | KEEP | No change |
| `gsd-prompt-guard.js` | PreToolUse | KEEP | No change |
| `gsd-statusline.js` | StatusLine | KEEP | No change |
| `gsd-workflow-guard.js` | PreToolUse | KEEP | No change |
| `claude-settings-guard.js` | PreToolUse | KEEP | No change |
| `hook-signal.js` | Shared utility | KEEP | No change |

### Forks: What Changes From the Source

| Component | Source | Key Changes in Fork |
|-----------|--------|---------------------|
| `executors/codex-exec.js` | `~/.claude/hooks/codex-exec.js` | (1) Replace absolute `./codex-pricing` require with `../lib/pricing`; (2) Remove `runWithFallback` — dispatch.js owns fallback; (3) Wrap `runCodexExec` as `execute(prompt, opts)` interface; (4) Add `available()` function |
| `executors/minimax-exec.js` | `~/.claude/hooks/minimax-exec.js` | (1) Replace absolute paths with relative requires; (2) Remove `runWithFallback` — dispatch.js owns fallback; (3) Replace absolute OPENAI_GLOBAL_PATH with lazy require pattern; (4) Wrap `runMinimax` as `execute(prompt, opts)` interface; (5) Add `available()` function |
| `lib/pricing.js` | `~/.claude/hooks/codex-pricing.js` | (1) Add six new model entries: gemini-3.1-pro, gemini-3-flash, sonnet-4.6, haiku-4.5, qwen-3.5-27b (free), perplexity-sonar; (2) Rename `CODEX_PRICING` to `MODEL_PRICING` |
| `hooks/token-logger.js` | `~/.claude/hooks/codex-token-logger.js` | (1) Change log path from `.planning/token-log.jsonl` to `.seraphim/token-log.jsonl`; (2) Expand record schema to handle all nine model names; (3) Update advisory context label |

### External Service Integration

| Service | Integration Pattern | Env Var | Notes |
|---------|--------------------|---------|----|
| Codex CLI | `spawn('codex', ['exec', '--full-auto', '--json', ...])` | `OPENAI_API_KEY` | 300s timeout; SIGTERM+SIGKILL pattern from existing codex-exec.js. Proven. |
| MiniMax API | OpenAI SDK `new OpenAI({ baseURL: 'https://api.minimax.io/v1', apiKey })` | `MINIMAX_API_KEY` | 90s timeout; temperature 0.01 (rejects 0); 3-retry exponential backoff. Proven. |
| Gemini API | `@google/generative-ai` npm package | `GEMINI_API_KEY` | Search grounding for Discover; thinking mode for Judge; function calling for internal discovery. Package not yet installed. |
| Qwen (ollama) | HTTP `POST localhost:11434/api/generate` | None (local) | 120s timeout; 32K context cap; JSON action wrapper for Forge; availability check via `/api/tags`. RTX 3090 in transit. |
| Perplexity | MCP (already configured, preferred) or `PERPLEXITY_API_KEY` HTTP | `PERPLEXITY_API_KEY` | MCP preferred — zero new deps. Paid account configured. |
| SearXNG | `curl 'http://localhost:8888/search?q=...&format=json'` in websearch.sh | None | For Codex/Qwen web access. Already running at localhost:8888. |
| Context7 | HTTP endpoint in fetchdocs.js | None | For Codex/Qwen documentation access. MCP already configured for Opus. |

### Internal Module Boundaries

| Boundary | Communication | Rule |
|----------|---------------|------|
| `commands/*.md` ↔ `dispatch.js` | Opus reads command, calls `node executors/dispatch.js phase prompt` via Bash | Commands are Markdown prompts — they instruct Opus what to do; Opus makes the Bash call |
| `dispatch.js` ↔ `executors/` | `require('./executor-id')` + standard interface calls only | dispatch.js never calls executor-specific methods; only `execute()`, `available()` |
| `executors/` ↔ `lib/pricing.js` | `require('../lib/pricing.js')` | No inline pricing anywhere in executors |
| `dispatch.js` ↔ `lib/config.js` | `require('../lib/config.js')` | dispatch.js never reads `.seraphim/config.json` directly |
| `commands/*.md` ↔ `lib/phase-state.js` | Opus calls `node lib/phase-state.js check phase-id` via Bash | Commands verify prerequisites before invoking dispatch |
| `tools/checkpoint.js` ↔ `dispatch.js` | `node executors/dispatch.js forge_checkpoint_static ...` subprocess | checkpoint.js triggers static review as a dispatch call; it does not call minimax-exec directly |
| `agents/*.md` ↔ `executors/` | Bash subprocess: `node executors/dispatch.js phase ...` inside agent | Agents that need non-Claude model output use dispatch.js via Bash, not direct executor require |

---

## Build Order and Dependencies

The mandated build order maps cleanly to dependency constraints:

```
GROUP A — No dependencies. Build immediately. All parallel.
  plugin.json
  config/profiles.json
  config/models.json
  lib/pricing.js         (fork + extend — no runtime deps)
  lib/config.js          (new — no executor deps)
  lib/phase-state.js     (new — no executor deps)
  tools/websearch.sh     (pure curl — no Node deps)
  tools/fetchdocs.js     (no executor deps)

GROUP B — Depends on lib/ only. Build after Group A. All parallel.
  executors/codex-exec.js      (fork + adapt; requires lib/pricing.js)
  executors/minimax-exec.js    (fork + adapt; requires lib/pricing.js)
  executors/gemini-exec.js     (new; requires lib/pricing.js)
  executors/qwen-exec.js       (new; requires lib/pricing.js)
  executors/perplexity-exec.js (new; no pricing needed — cost is flat)
  hooks/token-logger.js        (fork + adapt; requires lib/pricing.js)

GROUP C — Depends on all five executors + lib/config.js. Single file.
  executors/dispatch.js        (new; requires all executors + config.js + pricing.js)

GROUP D — Depends on dispatch.js. Build after Group C. Mostly parallel.
  tools/checkpoint.js          (depends on dispatch.js for static review phase)
  hooks/session-start.js       (depends on lib/config.js + lib/phase-state.js)
  commands/discover.md         (depends on dispatch.js being callable)
  commands/envision.md         (depends on dispatch.js)
  commands/judge.md            (depends on dispatch.js)
  commands/architect.md        (depends on dispatch.js)
  commands/forge.md            (depends on dispatch.js + checkpoint.js)
  commands/crucible.md         (depends on dispatch.js)
  commands/set-profile.md      (depends on lib/config.js)
  commands/show-profile.md     (depends on lib/config.js)
  commands/override.md         (depends on lib/config.js)
  commands/status.md           (depends on lib/phase-state.js)
  commands/new-project.md      (depends on lib/config.js)
  commands/run.md              (depends on all six phase commands)

GROUP E — Depends on corresponding commands. All parallel.
  agents/seraphim-envision.md
  agents/seraphim-judge.md
  agents/seraphim-architect.md
  agents/seraphim-crucible.md
  agents/seraphim-checkpoint.md

GROUP F — Forge checkpoint system integration
  Forge between-task loop logic (wired into forge.md command + checkpoint.js)
  Depends on: forge.md, checkpoint.js, dispatch.js, lib/phase-state.js
  This is the most complex integration; build and test in isolation.

GROUP G — Adaptive intelligence (build last)
  Decision logging in dispatch.js (extend Group C)
  decisions.jsonl schema documentation
  Analysis script for reading decisions.jsonl
  Dashboard extension (per-phase performance heatmap, profile cost comparison)
  Depends on: complete pipeline from Groups A–F; needs real decision data to test
```

### Dependency Graph (critical path)

```
lib/ (A) → executors/ (B) → dispatch.js (C) → commands/ (D) → agents/ (E)
                                              ↘ checkpoint.js (D)
                                              ↘ Forge loop (F)
                                              ↘ Adaptive intel (G)
```

### What Can Be Built in Parallel

Within each group, all components are independent and can be built simultaneously:
- All five executors in Group B share no state with each other
- All twelve commands in Group D share no state with each other
- All five agents in Group E share no state with each other

The only strict sequencing is: A before B, B before C, C before D, D before E, E+C before F, F before G.

---

## Anti-Patterns

### Anti-Pattern 1: Absolute Paths to ~/.claude/hooks/ in Plugin Executors

**What people do:** Copy the existing hooks verbatim, leaving absolute paths like `/home/alucard/.claude/hooks/codex-pricing` and `/home/alucard/.npm-global/lib/node_modules/openai` in the forked files.

**Why it's wrong:** Breaks on any machine that is not the original developer's workstation. The plugin must work from `~/.claude/plugins/seraphim/` regardless of username or global npm path.

**Do this instead:** Use `path.join(__dirname, '../lib/pricing.js')` for intra-plugin requires. For the OpenAI SDK, use the lazy-require pattern already present in minimax-exec.js: `try { require('openai') } catch { require(GLOBAL_PATH) }` — but replace the hardcoded GLOBAL_PATH with a resolved path using `require.resolve`.

### Anti-Pattern 2: Fallback Chain Logic Inside Individual Executors

**What people do:** The existing `runWithFallback()` in minimax-exec.js embeds the Codex → MiniMax → user escalation logic inside the executor. Fork it into the plugin unchanged.

**Why it's wrong:** In Seraphim, fallback chains are profile-specific. The Budget profile's forge fallback is Qwen, not MiniMax. Embedding fallback in an executor hardcodes assumptions that belong in dispatch.js.

**Do this instead:** Remove `runWithFallback()` from the forked minimax-exec.js. dispatch.js reads `profile.fallbackChain[phase]` and tries each executor in order via `available()` + `execute()`.

### Anti-Pattern 3: Phase Commands That Do Their Own Config Resolution

**What people do:** Each command reads `.seraphim/config.json` directly and resolves which model to call inline.

**Why it's wrong:** Config resolution logic duplicated across twelve command files. Adding a new profile or changing fallback behavior requires updating every command.

**Do this instead:** Commands invoke `node executors/dispatch.js [phase] [prompt]` via Bash. dispatch.js owns all config resolution. Commands never touch config directly.

### Anti-Pattern 4: Running Forge Without Phase-State Guard

**What people do:** Allow `/seraphim:forge [id]` to run even if Architect has not produced `blueprint.md`.

**Why it's wrong:** Forge reads the blueprint. Without it, Codex gets an empty or missing prompt and burns subscription quota producing nothing useful.

**Do this instead:** Every phase command checks `phase-state.js` for the predecessor's completion before invoking dispatch.js. `forge.md` verifies `blueprint.md` exists and is non-empty. `/seraphim:run` enforces the full sequence automatically.

### Anti-Pattern 5: Leaving Seven Retiring Hooks Active During and After Migration

**What people do:** Register the new plugin hooks in settings.json while leaving the seven retiring hooks (codex-plan-reviewer, minimax-post-scan, codex-review-gate, etc.) also registered.

**Why it's wrong:** The Stop hook chain becomes four expensive model calls on every session end — double-reviewing work that the pipeline already reviewed. MiniMax adversarial review fires on every Write/Edit in addition to the Crucible phase.

**Do this instead:** Migration is an atomic settings.json edit: register plugin hooks AND remove retiring hooks in the same operation. Do not leave both active simultaneously even for testing.

### Anti-Pattern 6: Storing Two Separate Token Logs and Not Updating the Global Aggregator

**What people do:** Fork token-logger.js to write to `.seraphim/token-log.jsonl` and leave `codex-global-aggregator.js` unchanged — it continues scanning for `.planning/token-log.jsonl` only.

**Why it's wrong:** The global dashboard stops including Seraphim sessions. Decision Intelligence panel only sees GSD data. The dashboard becomes misleading.

**Do this instead:** Extend `codex-global-aggregator.js` (or its Seraphim-aware replacement) to also scan for `.seraphim/token-log.jsonl` files. The global aggregation step happens in Group G (adaptive intelligence phase).

---

## Migration Path from Hooks to Plugin

### Step 1: Plugin scaffold (no behavior change)

Create `~/.claude/plugins/seraphim/` with `plugin.json`. Claude Code picks up the plugin — the `/seraphim:` command namespace appears. No hooks are modified. Existing v2.0 behavior continues unchanged.

### Step 2: Fork and adapt executors + lib/

Build all of Group A and Group B. Executors exist in the plugin but nothing invokes them yet. Old hooks still run unchanged.

### Step 3: Build dispatch.js + commands + agents

Build Group C, D, and E. Phase commands become invokable. The six-phase pipeline is functional but the old hooks still also run.

### Step 4: Atomic hook migration (single settings.json edit)

In one edit to `~/.claude/settings.json`:

**Remove from Stop chain:**
- `codex-review-gate.js`
- `codex-plan-reviewer.js`
- `codex-multi-round-reviewer.js`
- `codex-superpowers-plan-reviewer.js`

**Remove from PostToolUse chain:**
- `minimax-post-scan.js`
- `minimax-compress.js`
- `codex-wave-validator.js`
- `codex-token-logger.js` (replaced by plugin fork)

**Remove from PreToolUse chain:**
- `codex-router.js`

**Add to SessionStart chain:**
- `~/.claude/plugins/seraphim/hooks/session-start.js`

**Add to PostToolUse chain:**
- `~/.claude/plugins/seraphim/hooks/token-logger.js`

**Keep unchanged:** gsd-*, decision-logger.js, codex-cost-reporter.js, codex-global-aggregator.js, claude-settings-guard.js, hook-signal.js

### Step 5: Forge checkpoint system (Group F)

Build checkpoint.js and wire the between-task loop into forge.md. Test in isolation with a simple blueprint before end-to-end testing.

### Step 6: Adaptive intelligence (Group G)

Add decision logging to dispatch.js. Build the analysis script. Extend the dashboard with the per-phase performance heatmap.

---

## Scaling Considerations

Seraphim is a local single-developer tool. Scaling concerns are about session cost, model availability, and decisions.jsonl growth — not server load.

| Concern | At current usage | If heavy usage (50+ cycles/day) |
|---------|-----------------|--------------------------------|
| Codex rate limit (Plus weekly cap) | dispatch.js detects via `isCodexRateLimited()` pattern from codex-exec.js; falls back per profile | Same detection; consider Moderate profile as default if Codex cap hits frequently |
| MiniMax pre-answer latency (~55s) | 90s timeout proven in minimax-exec.js; fire-and-forget for adversarial passes | Acceptable; MiniMax used only for Judge/Crucible, not every call |
| Qwen local inference (120s) | RTX 3090 not yet available; Balanced/Budget profiles blocked until GPU arrives | Once available, ollama context limit (32K) is the practical constraint |
| decisions.jsonl growth | ~10-30 records per pipeline cycle; 1MB/day at heavy usage | After ~10K records add mtime-gated incremental read (same pattern as codex-global-aggregator.js) |
| Runaway feedback loops | Hard caps in phase-state.js (max 2 per loop type) enforced before re-invoking | Hard caps are non-negotiable; raising them requires explicit config change |
| Cost overrun | decisions.jsonl tracks per-call cost; session-start.js reports running total | $15/day budget enforced by human review of session-start report; no auto-spend guard needed |

---

## Sources

- Direct inspection: `/home/alucard/.claude/hooks/codex-exec.js` (v1.1.0) — executor pattern, JSONL parsing, SIGTERM+SIGKILL timeout
- Direct inspection: `/home/alucard/.claude/hooks/minimax-exec.js` (v1.1.0) — OpenAI SDK baseURL swap, retry backoff, runWithFallback pattern to retire
- Direct inspection: `/home/alucard/.claude/hooks/codex-pricing.js` (v1.0.0) — pricing table structure to fork and extend
- Direct inspection: `/home/alucard/.claude/hooks/codex-token-logger.js` (v1.0.0) — JSONL append pattern, [CODEX_RESULT] marker approach
- Direct inspection: `/home/alucard/.claude/hooks/codex-router.js` (v2.0.0) — PreToolUse advisory pattern, config resolution
- Direct inspection: `/home/alucard/.claude/hooks/codex-wave-validator.js` (v2.0.0) — PostToolUse checkpoint pattern
- Direct inspection: `/home/alucard/.claude/settings.json` — all hook registrations, event types, timeout values, enabled plugins
- Direct inspection: `/home/alucard/.claude/plugins/marketplaces/claude-plugins-official/plugins/example-plugin/` — plugin file structure (commands/, agents/ layout)
- Direct inspection: `/home/alucard/.claude/plugins/marketplaces/claude-plugins-official/plugins/code-simplifier/agents/code-simplifier.md` — agent frontmatter format (name, description, model)
- Direct inspection: `/home/alucard/.claude/plugins/marketplaces/claude-plugins-official/plugins/claude-code-setup/skills/claude-automation-recommender/references/plugins-reference.md` — plugin registration pattern
- Direct inspection: `docs/specs/2026-04-04-seraphim-v3-design.md` — approved design spec (primary source)
- Direct inspection: `.planning/PROJECT.md` — project constraints, key decisions, requirements

---

*Architecture research for: Seraphim v3.0 — six-phase multi-model Claude Code plugin*
*Researched: 2026-04-04*
