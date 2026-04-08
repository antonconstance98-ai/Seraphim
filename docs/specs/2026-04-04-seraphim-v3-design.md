# Seraphim v3.0 — Six-Phase Multi-Model Creative Pipeline

**Date:** 2026-04-04
**Status:** Approved design, ready for implementation planning
**Scope:** Full milestone (v3.0)

## Overview

Seraphim is a standalone Claude Code plugin that implements a six-phase creative pipeline where each phase is owned by the optimal AI model for that cognitive task. Five profiles (Performance, Balanced, Moderate, Budget, Frugal) shift model assignments across the same pipeline structure. Built-in adaptive intelligence tracks every decision and learns which models perform best for which tasks over time.

Seraphim is a hard fork from GSD (Get Shit Done). It inherits the project management foundation (milestones, roadmaps, stats) but replaces the four-phase workflow (research, plan, execute, verify) with a six-phase pipeline designed from first principles.

## Core Principles

- **Six wings, six phases** — Discover, Envision, Judge, Architect, Forge, Crucible
- **One model per phase per profile** — no ambiguity about who owns what
- **The human is the orchestrator** — decides which profile, which project, when to advance
- **Claude Opus is the session host** — reads phase outputs and routes to the next model, but doesn't necessarily do every phase
- **Adaptive cost** — same quality pipeline, five price points
- **Not code-only** — works for research, writing, science, any creative project
- **Models are swappable** — the phase structure is permanent, model assignments are config

## The Six Phases

### I. DISCOVER
*"What exists? What's possible? What's been tried?"*

Two parallel tracks:
- **External**: Web research — docs, prior art, benchmarks, gotchas, community knowledge
- **Internal**: Codebase/project exploration — existing patterns, file structure, dependencies, what will break

Both tracks write raw findings to `.seraphim/phases/{N}/discovery/`. No synthesis yet — just raw material. The Architect phase synthesizes.

For non-code projects: External still searches the web. Internal reads project files, notes, prior drafts, data sets.

Output:
- `.seraphim/phases/{N}/discovery/external.md`
- `.seraphim/phases/{N}/discovery/internal.md`

### II. ENVISION
*"What could we create?"*

Receives discovery output. Generates 3-5 distinct approaches with trade-offs. No commitment — pure divergent thinking. Each approach gets a name, a one-paragraph description, pros, cons, and a "gut feeling" rating.

For non-code projects: Approaches might be thesis angles, experimental designs, narrative structures, architectural patterns.

Output: `.seraphim/phases/{N}/vision.md`

### III. JUDGE
*"What could go wrong?"*

Receives the vision output. Tries to destroy every approach. Finds fatal flaws, missing requirements, scalability problems, security holes, logical contradictions. Ranks the survivors. May declare "none survive" and send back to Envision.

Output: `.seraphim/phases/{N}/judgment.md` — each approach marked SURVIVES / FATAL FLAW / CONDITIONAL (with conditions)

**Loop:** If zero approaches survive, Envision runs again with Judge's feedback. Max 2 loops.

### IV. ARCHITECT
*"How exactly do we build the survivor?"*

Takes the top-ranked surviving approach and creates a detailed blueprint. Task breakdown, dependencies, interfaces, sequence, file paths, test criteria. This is the only phase that must be rigorous and precise.

Output: `.seraphim/phases/{N}/blueprint.md` — concrete enough that the Forge phase needs zero creative decisions.

### V. FORGE
*"Make it real."*

Executes the blueprint task by task. Between tasks: automated checkpoint (runtime sanity + static code review). On failure: retry with feedback, max 2 retries per task before escalating.

For non-code projects: "Forge" means writing prose, generating data, running experiments.

Output: Git commits + `.seraphim/phases/{N}/forge-log.md`

**Between-task checkpoint:**
1. Runtime check — tests pass, lint clean, imports resolve
2. Static review — diff analyzed for bugs, security, logic errors
3. Both pass → next task
4. Either flags issue → Forge retries with feedback (max 2)

### VI. CRUCIBLE
*"Try to destroy it. Polish what survives."*

**Pass 1 — Verification:** Does the output match the blueprint? Goal-backward check against every requirement.

**Pass 2 — Adversarial:** Actively try to break it. Edge cases, security, performance, logic errors, missing handling.

Output: `.seraphim/phases/{N}/crucible.md` — PASS / FAIL with detailed findings. Fail sends back to Forge with specific fix instructions. Max 2 loops.

## Model Roster

Nine models available for assignment across phases:

| Model | Access Method | Cost (per Mtok in/out) | Superpower |
|-------|--------------|------------------------|------------|
| Claude Opus 4.6 | Native session | $5/$25 | Best reasoning, creativity, architecture |
| Claude Sonnet 4.6 | Subagent | $3/$15 | Reliable all-rounder with full tool use |
| Claude Haiku 4.5 | Subagent | $0.80/$4 | Fast, cheap utility tasks |
| Codex GPT-5.4 | CLI full-auto | $0 (Plus sub) | Autonomous code execution, sandboxed |
| MiniMax M-2.7 | API | $0.30/$1.20 | Bug scanning, adversarial review, 100% Opus-match on bug detection |
| Gemini 3.1 Pro Preview | API | $2/$12 | Agentic planning, multi-step tool use, 1M context |
| Gemini 3 Flash | API | $0.50/$3 | Fast reasoning (90.4% GPQA), search grounding |
| Qwen 3.5-27B | Local (ollama) | $0 (local GPU) | Free, 95% instruction following, strong coding |
| Perplexity Sonar | MCP / API | Paid account | Live web research with citations |

## Five Profiles

### PERFORMANCE
*Every phase gets the best model for that cognitive task. Cost: $3-8 per cycle.*

| Phase | Model | Mechanism |
|-------|-------|-----------|
| DISCOVER (external) | Perplexity Sonar | MCP tools |
| DISCOVER (internal) | Gemini 3.1 Pro Preview | API + tool wrappers |
| ENVISION | Claude Opus 4.6 | Native session |
| JUDGE | Gemini 3 Flash | API |
| ARCHITECT | Claude Opus 4.6 | Native session |
| FORGE | Codex GPT-5.4 | CLI full-auto |
| FORGE checkpoint (runtime) | Sonnet 4.6 | Subagent |
| FORGE checkpoint (static) | MiniMax M-2.7 | API |
| CRUCIBLE (verify) | Claude Opus 4.6 | Native session |
| CRUCIBLE (adversarial) | MiniMax M-2.7 | API |

### BALANCED (requires local LLM)
*Opus only architects. Qwen fills gaps. Cost: $1-3 per cycle.*

| Phase | Model | Mechanism |
|-------|-------|-----------|
| DISCOVER (external) | Perplexity Sonar | MCP tools |
| DISCOVER (internal) | Qwen 3.5-27B | Local (ollama) |
| ENVISION | Qwen 3.5-27B | Local (ollama) |
| JUDGE | MiniMax M-2.7 | API |
| ARCHITECT | Claude Opus 4.6 | Native session |
| FORGE | Codex GPT-5.4 | CLI full-auto |
| FORGE checkpoint (runtime) | Qwen 3.5-27B | Local (ollama) |
| FORGE checkpoint (static) | MiniMax M-2.7 | API |
| CRUCIBLE (verify) | Qwen 3.5-27B | Local (ollama) |
| CRUCIBLE (adversarial) | MiniMax M-2.7 | API |

### MODERATE (cloud-only)
*Opus only architects. Cloud models fill the rest. Cost: $1.50-4 per cycle.*

| Phase | Model | Mechanism |
|-------|-------|-----------|
| DISCOVER (external) | Perplexity Sonar | MCP tools |
| DISCOVER (internal) | Gemini 3 Flash | API |
| ENVISION | Gemini 3.1 Pro Preview | API |
| JUDGE | MiniMax M-2.7 | API |
| ARCHITECT | Claude Opus 4.6 | Native session |
| FORGE | Codex GPT-5.4 | CLI full-auto |
| FORGE checkpoint (runtime) | Sonnet 4.6 | Subagent |
| FORGE checkpoint (static) | MiniMax M-2.7 | API |
| CRUCIBLE (verify) | Gemini 3 Flash | API |
| CRUCIBLE (adversarial) | MiniMax M-2.7 | API |

### BUDGET (requires local LLM)
*Opus architects, Qwen does everything else. Cost: $0.50-1.50 per cycle.*

| Phase | Model | Mechanism |
|-------|-------|-----------|
| DISCOVER (external) | Gemini 3 Flash | API |
| DISCOVER (internal) | Qwen 3.5-27B | Local (ollama) |
| ENVISION | Qwen 3.5-27B | Local (ollama) |
| JUDGE | Qwen 3.5-27B | Local (ollama) |
| ARCHITECT | Claude Opus 4.6 | Native session |
| FORGE | Qwen 3.5-27B | Local (ollama + wrapper) |
| FORGE checkpoint (runtime) | Qwen 3.5-27B | Local (ollama) |
| FORGE checkpoint (static) | Qwen 3.5-27B | Local (ollama) |
| CRUCIBLE (verify) | Qwen 3.5-27B | Local (ollama) |
| CRUCIBLE (adversarial) | Qwen 3.5-27B | Local (ollama) |

### FRUGAL (cloud-only, minimum spend)
*Opus architects, cheapest cloud models everywhere else. Cost: $0.80-2 per cycle.*

| Phase | Model | Mechanism |
|-------|-------|-----------|
| DISCOVER (external) | Gemini 3 Flash | API |
| DISCOVER (internal) | Claude Haiku 4.5 | Subagent |
| ENVISION | MiniMax M-2.7 | API |
| JUDGE | MiniMax M-2.7 | API |
| ARCHITECT | Claude Opus 4.6 | Native session |
| FORGE | Codex GPT-5.4 | CLI full-auto |
| FORGE checkpoint (runtime) | Claude Haiku 4.5 | Subagent |
| FORGE checkpoint (static) | MiniMax M-2.7 | API |
| CRUCIBLE (verify) | Claude Haiku 4.5 | Subagent |
| CRUCIBLE (adversarial) | MiniMax M-2.7 | API |

## Profile Configuration

Profiles are presets, not locks. Every phase-model assignment is individually overridable.

### Config file: `.seraphim/config.json`

```json
{
  "profile": "moderate",
  "opus_enabled": true,
  "overrides": {
    "architect": "gemini-3.1-pro"
  }
}
```

### Toggle behaviors

| Setting | Effect |
|---------|--------|
| `opus_enabled: false` | Opus phases fall back to profile's next-best model |
| `overrides: { "architect": "sonnet" }` | Just that one phase changes |
| `profile: "custom"` | All phases explicitly defined by user |

### Opus-disabled fallbacks

| Profile | Opus falls back to |
|---------|-------------------|
| PERFORMANCE | Gemini 3.1 Pro Preview |
| BALANCED | Qwen 3.5-27B |
| MODERATE | Gemini 3.1 Pro Preview |
| BUDGET | Qwen 3.5-27B |
| FRUGAL | Gemini 3 Flash |

### Full custom example

```json
{
  "profile": "custom",
  "opus_enabled": false,
  "phases": {
    "discover_external": "perplexity-sonar",
    "discover_internal": "qwen-3.5-27b",
    "envision": "gemini-3.1-pro",
    "judge": "gemini-3-flash",
    "architect": "sonnet-4.6",
    "forge": "codex-gpt-5.4",
    "forge_checkpoint_runtime": "qwen-3.5-27b",
    "forge_checkpoint_static": "minimax-m2.7",
    "crucible_verify": "qwen-3.5-27b",
    "crucible_adversarial": "minimax-m2.7"
  }
}
```

## Technical Architecture

### Plugin Structure

```
~/.claude/plugins/seraphim/
├── plugin.json                    # Plugin manifest
├── config/
│   ├── profiles.json              # Five preset profiles + model registry
│   └── models.json                # Model definitions (mechanism, pricing, capabilities)
├── commands/
│   ├── discover.md                # /seraphim:discover
│   ├── envision.md                # /seraphim:envision
│   ├── judge.md                   # /seraphim:judge
│   ├── architect.md               # /seraphim:architect
│   ├── forge.md                   # /seraphim:forge
│   ├── crucible.md                # /seraphim:crucible
│   ├── run.md                     # /seraphim:run (all six phases)
│   ├── set-profile.md             # /seraphim:set-profile
│   ├── show-profile.md            # /seraphim:show-profile
│   ├── override.md                # /seraphim:override
│   ├── status.md                  # /seraphim:status
│   └── new-project.md             # /seraphim:new-project
├── agents/
│   ├── seraphim-envision.md       # Envision phase agent
│   ├── seraphim-judge.md          # Judge phase agent
│   ├── seraphim-architect.md      # Architect phase agent
│   ├── seraphim-crucible.md       # Crucible phase agent
│   └── seraphim-checkpoint.md     # Between-task checkpoint agent
├── executors/
│   ├── codex-exec.js              # Codex CLI wrapper (fork from existing)
│   ├── minimax-exec.js            # MiniMax API wrapper (fork from existing)
│   ├── gemini-exec.js             # Gemini API wrapper (new)
│   ├── qwen-exec.js               # Local Qwen via ollama (new)
│   ├── perplexity-exec.js         # Perplexity via MCP bridge (new)
│   └── dispatch.js                # Reads config, routes phase -> executor
├── tools/
│   ├── websearch.sh               # SearXNG wrapper for Codex/Qwen
│   ├── fetchdocs.js               # Context7 HTTP wrapper for Codex/Qwen
│   └── checkpoint.js              # Test runner + lint checker
├── hooks/
│   ├── session-start.js           # Load profile, report status
│   └── token-logger.js            # Unified multi-model cost tracking
└── lib/
    ├── pricing.js                 # All nine models' pricing
    ├── config.js                  # Read/write .seraphim/config.json
    └── phase-state.js             # Track phase progress per project
```

### Per-Project State

```
<project>/.seraphim/
├── config.json                    # Profile + overrides for this project
├── token-log.jsonl                # All model calls, costs, tokens
├── decisions.jsonl                # Routing decisions for adaptive learning
└── phases/
    └── 01-feature-name/
        ├── discovery/
        │   ├── external.md
        │   └── internal.md
        ├── vision.md
        ├── judgment.md
        ├── blueprint.md
        ├── forge-log.md
        └── crucible.md
```

### Dispatch Flow

Central piece is `dispatch.js` — reads the profile, resolves overrides, and routes to the right executor:

```
User invokes /seraphim:forge
  -> command reads .seraphim/config.json
  -> profile "balanced" + no overrides -> forge = "codex-gpt-5.4"
  -> dispatch.js loads codex-exec.js
  -> codex-exec.js spawns `codex exec --full-auto --json`
  -> output parsed, git diff captured
  -> checkpoint.js runs (tests + static review via profile's checkpoint models)
  -> pass? -> next task
  -> fail? -> codex-exec.js retries with feedback
```

### Model Executor Interface

Every executor implements the same interface:

```javascript
module.exports = {
  execute(prompt, opts)  // -> { success, output, tokens, cost, error }
  stream(prompt, opts)   // -> ReadableStream (where supported)
  available()            // -> boolean (can this model run right now?)
}
```

Swap Gemini for Qwen? Change one line in config. The dispatch layer doesn't care which model it's talking to.

### Qwen Local Execution Wrapper

Qwen runs via ollama HTTP API at `localhost:11434`:

- Startup detection: check if ollama is running, start if not
- Model loading: `ollama pull qwen3.5:27b-q4` on first use
- Context management: cap at 32K tokens on consumer hardware
- Tool simulation for Forge mode: Qwen outputs structured JSON instructions (`{"action": "write", "path": "...", "content": "..."}`), wrapper executes them on disk
- Timeout: 120s default (local inference is slower)

### Gemini API Integration

Uses `@google/generative-ai` npm package:

- Auth: `GEMINI_API_KEY` env var
- Search grounding: enabled for Discover phase
- Function calling: enabled for internal discovery
- Thinking mode: enabled for Judge phase

### Helper Scripts for Non-Claude Models

Models running outside Claude Code (Codex CLI, Qwen, Gemini) can't use MCP tools. Two helper scripts bridge the gap:

- `websearch.sh` — wraps SearXNG at localhost:8888, returns JSON results
- `fetchdocs.js` — calls context7 HTTP endpoint directly, returns documentation

### Feedback Loops

| Trigger | Loop |
|---------|------|
| Judge kills all approaches | Envision re-runs with Judge feedback (max 2 loops) |
| Forge task fails checkpoint | Forge retries with feedback (max 2 retries per task) |
| Crucible fails verification | Forge re-runs failed tasks (max 2 loops) |
| Crucible finds adversarial issues | Forge patches specific issues (max 2 loops) |

All loops have hard caps. Max loops exceeded -> stop and surface to the human.

### Phase Transitions

Phases advance explicitly or via full-auto:

```
/seraphim:discover 1        # runs Discover for phase 1
/seraphim:envision 1        # runs Envision (reads discovery output)
/seraphim:judge 1            # runs Judge (reads vision output)
/seraphim:architect 1        # runs Architect (reads judgment output)
/seraphim:forge 1            # runs Forge (reads blueprint)
/seraphim:crucible 1         # runs Crucible (reads forge output)

/seraphim:run 1              # all six in sequence, auto-advancing
/seraphim:run 1 --from judge # resume from Judge phase
```

## Hooks Consolidation

These existing hooks become redundant (consolidated into the pipeline):

| Hook | Replaced By |
|------|------------|
| codex-review-gate.js | Crucible phase |
| codex-plan-reviewer.js | Judge phase |
| codex-multi-round-reviewer.js | Judge phase |
| minimax-post-scan.js | Crucible adversarial pass |
| minimax-compress.js | Each executor manages its own context |
| codex-router.js | dispatch.js |
| codex-wave-validator.js | Forge checkpoint logic |

Kept (forked into plugin):
| Hook | Why |
|------|-----|
| token-logger.js | Unified cost tracking across all nine models |
| session-start.js | Profile loading + status report |

## Adaptive Intelligence

The v3.0 "Adaptive Intelligence" goal is built into Seraphim natively:

### Data Collection

Every phase execution logs to `decisions.jsonl`:
- Which model was used, which phase, which profile
- Tokens consumed, cost, latency
- Outcome: success/failure, retry count, loop count
- Quality signals: did Crucible find issues? Did Judge kill approaches?

### Learning

Periodic analysis of `decisions.jsonl` to answer:
- Which models produce the fewest Crucible failures per phase?
- Which profiles have the best cost/quality ratio?
- Are certain models consistently flagged by downstream phases?
- Which checkpoint reviewers catch the most real issues vs false positives?

### Auto-Adjustment

System can recommend profile/override changes based on accumulated data:
- "Qwen's Envision output was rejected by Judge in 4/5 runs — consider upgrading to Gemini 3.1 Pro for Envision"
- "MiniMax adversarial pass hasn't found a real issue in 20 runs — consider reducing frequency"
- Safety guardrails: recommendations require human approval, never auto-applied

### Insights Dashboard

Extends existing dashboard (`~/.claude/dashboard/dashboard.html`) with:
- Per-phase model performance heatmap
- Profile cost comparison over time
- Recommendation log (what the system suggested, what the user accepted/rejected)

## Non-Code Project Support

The pipeline is project-type agnostic. The blueprint from Architect declares the project type:

| Project Type | Forge Action | Crucible Action |
|-------------|-------------|----------------|
| Code | Edit files, run tests, commit | Run tests + adversarial code review |
| Research/Writing | Generate prose, citations, structure | Fact-check + argument stress-test |
| Science | Run analysis scripts, generate data | Replicate results + methodology review |
| Mixed | Code + prose | Both verification approaches |

## What Gets Forked vs Built New

| Component | Source | Action |
|-----------|--------|--------|
| codex-exec.js | Existing ~/.claude/hooks/codex-exec.js | Fork + adapt |
| minimax-exec.js | Existing ~/.claude/hooks/minimax-exec.js | Fork + adapt |
| pricing.js | Existing ~/.claude/hooks/codex-pricing.js | Fork + extend (Gemini, Qwen, Perplexity) |
| token-logger.js | Existing ~/.claude/hooks/codex-token-logger.js | Fork + adapt |
| gemini-exec.js | New | Google AI SDK integration |
| qwen-exec.js | New | Ollama HTTP API wrapper |
| perplexity-exec.js | New | MCP bridge or direct API |
| dispatch.js | New | Central router |
| websearch.sh | New | SearXNG curl wrapper |
| fetchdocs.js | New | Context7 HTTP wrapper |
| Phase workflows | Conceptual inspiration from GSD | Rewrite from scratch |
| Agents | Conceptual inspiration from GSD | Rewrite from scratch |

## Environment Requirements

| Requirement | Status |
|-------------|--------|
| Node.js v22+ | Installed |
| Codex CLI v0.118.0+ | Installed |
| OpenAI SDK v6.33.0+ | Installed |
| ollama + Qwen 3.5-27B Q4 | Pending (RTX 3090 in transit) |
| Google AI SDK (@google/generative-ai) | To install |
| GEMINI_API_KEY | To configure |
| MINIMAX_API_KEY | Configured |
| OPENAI_API_KEY | Configured |
| Perplexity MCP | Configured |
| SearXNG at localhost:8888 | Running |
| RTX 3090 GPU | In transit |

---

*Design approved 2026-04-04. Ready for implementation planning.*
