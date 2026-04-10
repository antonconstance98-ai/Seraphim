# Seraphim

## What This Is

A standalone Claude Code plugin that implements a six-phase creative pipeline (Discover, Envision, Judge, Architect, Forge, Crucible) where each phase is owned by the optimal AI model for that cognitive task. Nine models are available across five cost profiles (Performance, Balanced, Moderate, Budget, Frugal). Built-in adaptive intelligence tracks every routing decision and learns which models perform best for which tasks over time.

Seraphim is a hard fork from GSD (Get Shit Done), evolved from the Claude X Codex multi-model integration project (v1.0-v2.0). It replaces hook-based model routing with a unified phase pipeline, and replaces the four-phase workflow with six phases designed from first principles.

## Core Value

Six wings, six phases, six cognitive tasks — each assigned to the model that does it best. The human orchestrates. AI converges. Adaptive intelligence makes the system smarter over time.

## Shipped: v1.0 Codex Integration (2026-03-31)

Hook-based integration of Codex GPT-5.4 into Claude Code. Advisory routing, plan review loops, token tracking, cost reporting.

## Shipped: v1.1 Global Dashboard (2026-04-01)

Global JSONL aggregation, per-project metrics, Chart.js dashboard, session history.

## Shipped: v2.0 Three-Model Intelligence (2026-04-03)

MiniMax M-2.7 integrated as third model. Dual review gate, adversarial plan review, PostToolUse bug scanning, context compression, execution pipeline with fallback chain, three-model cost reporting.

## Shipped: v3.0 Seraphim (2026-04-09)

Six-phase creative pipeline (Discover→Crucible) with 9 models, 5 profiles, adaptive intelligence, multi-project dashboard (Vercel), human-AI task routing, and local RAG. 13 phases, 43 plans, 21 slash commands.

## Shipped: v3.1 Seraphim Project Management (2026-04-09)

Full project management layer: roadmaps with milestone-feature hierarchy, feature queues with WIP limits and dependency guards, human task inbox (decision/research/review/validation/skills), pause/resume PM context, milestone archival with cost attribution, cross-project overview terminal command, PM data sync to Neon, dashboard PM panels (roadmap tree, task management, cross-project overview with sparklines). 3 phases, 12 plans, 30 requirements.

## Shipped: v3.2 Idea-to-Shipped Journey (2026-04-10)

Full idea-to-shipped development workflow as native Seraphim commands. 7 phases, 26 plans, 69 requirements. Key deliverables:
- Seed capture, requirements definition (REQ-IDs), phased roadmaps with waves and dependencies
- Wave-structured planning with deep task specs, parallel execution with worktree isolation
- Research system (scope + run commands), session management (pause/resume/report)
- Navigation routing (next/do/progress), codebase mapping with parallel agents
- Phase lifecycle management (add/insert/remove), unified workflow settings
- Human task inbox, systematic debugging with persistent state, forensics
- Pipeline redesign: Discover (interactive round-loop), Envision (convergent vision), Judge (two-pass vision+architecture), Architect (N technical plans), Forge (GSD-style per-phase cycle)
- Goal-backward verification, conversational UAT, Nyquist gap auditing
- Dashboard progress panels, 7-day velocity chart, roadmap tree, milestone auditing
- Automated test suite: 65 tests covering all pipeline commands and dashboard components

**Previous target features (v3.0):**
- Standalone Claude Code plugin at `~/.claude/plugins/seraphim/`
- Six phases: Discover, Envision, Judge, Architect, Forge, Crucible
- Nine model integrations: Opus 4.6, Sonnet 4.6, Haiku 4.5, Codex GPT-5.4, MiniMax M-2.7, Gemini 3.1 Pro, Gemini 3 Flash, Qwen 3.5-27B (local), Perplexity Sonar
- Five profiles: Performance, Balanced, Moderate, Budget, Frugal
- Per-phase model overrides and opus_enabled toggle
- Unified executor interface (dispatch.js routes to model-specific executors)
- Helper scripts for non-Claude models (websearch.sh, fetchdocs.js)
- Between-task checkpoints in Forge phase (runtime + static review)
- Feedback loops with hard caps (Judge->Envision, Crucible->Forge)
- Adaptive intelligence: decision logging, pattern learning, auto-recommendations
- Consolidated hooks (remove 7 redundant hooks, keep token logger + session start)
- Works for code, research, writing, science, and mixed projects

## Current State

**v3.2 shipped 2026-04-10.** Full idea-to-shipped workflow with 40+ slash commands, redesigned 6-phase pipeline, verification system, and dashboard visualization. 65-test automated verification suite.

**Tech debt (from v3.2 audit):** Pre-existing TypeScript error in dashboard `ingest/route.ts`. Residual v3.1 items: Neon DDL pending, ARIA gaps, no error boundaries, cost_usd stub.

## Requirements

### Validated

- ✓ Claude config hooks auto-route simple/defined tasks to Codex CLI — *v1.0*
- ✓ Built-in token usage tracking logs every model call with tokens used, cost, and task type — *v1.0*
- ✓ Opus remains the sole model for architectural decisions and complex reasoning tasks — *v1.0*
- ✓ GSD plugin modified to use Codex at specific workflow points — *v1.0*
- ✓ Codex CLI preferred over API calls wherever possible — *v1.0*
- ✓ Superpowers plugin modified to use Codex for parallel hypothesis testing — *v1.0*
- ✓ Opus-Codex plan review loop (2 rounds) triggers before every plan — *v1.0*
- ✓ OpenAI API used only for quick model-to-model communication — *v1.0*
- ✓ Token tracking generates session reports showing savings vs Opus-only baseline — *v1.0*
- ✓ Global token log aggregator scans all projects for token-log.jsonl files — *v1.1*
- ✓ Per-project metrics breakdown (costs, savings, tokens, models, review catch rates) — *v1.1*
- ✓ Time trend charts showing costs and savings over time (daily/weekly) — *v1.1*
- ✓ Session history listing recent sessions per project with individual stats — *v1.1*
- ✓ Self-contained HTML dashboard at ~/.claude/dashboard/ (inline CSS/JS, no server) — *v1.1*
- ✓ SessionStart hook auto-regenerates the dashboard on every session — *v1.1*
- ✓ MiniMax M-2.7 SDK module (minimax-exec.js) with OpenAI SDK baseURL swap, zero new deps — *v2.0*
- ✓ MiniMax pricing added to codex-pricing.js; Opus 4.6 pricing corrected ($5/$25) — *v2.0*
- ✓ Dual review gate: Codex + MiniMax run in parallel on Stop hook, BLOCK if either flags — *v2.0*
- ✓ MiniMax serves as adversarial reviewer (Round 2) in plan review — *v2.0*
- ✓ PostToolUse MiniMax bug/security scanner after every Write/Edit — *v2.0*
- ✓ Codex execution pipeline: gsd-executor generates handoff specs, Codex CLI writes code — *v2.0*
- ✓ Fallback chain: Codex CLI → MiniMax API → prompt user (fail-closed) — *v2.0*
- ✓ Universal context compression via MiniMax (large diffs, files, tool outputs) — *v2.0*
- ✓ Token tracking and cost reporting updated for three models — *v2.0*
- ✓ Global dashboard updated to show three-model metrics and Fallback Events — *v2.0*

### Active (v3.0)

- Standalone Seraphim plugin at `~/.claude/plugins/seraphim/` with plugin.json manifest
- Six phase workflows: Discover, Envision, Judge, Architect, Forge, Crucible
- Unified executor interface with dispatch.js routing to model-specific executors
- Nine model executors: codex-exec.js, minimax-exec.js, gemini-exec.js, qwen-exec.js, perplexity-exec.js, plus Claude subagents
- Five profile presets (Performance, Balanced, Moderate, Budget, Frugal) in profiles.json
- Per-phase model overrides via .seraphim/config.json
- opus_enabled toggle with per-profile fallback chain
- Helper scripts (websearch.sh, fetchdocs.js) for non-Claude model tool access
- Between-task checkpoint system in Forge phase (runtime + static review)
- Feedback loops: Judge->Envision (max 2), Crucible->Forge (max 2)
- Decision logging to decisions.jsonl for adaptive intelligence
- Pattern learning engine analyzing model performance per phase
- Auto-recommendation system with human-approval guardrails
- Consolidated hooks: remove 7 redundant hooks from ~/.claude/hooks/
- Insights dashboard extension with per-phase model performance heatmap
- Non-code project support (research, writing, science, mixed)
- Qwen 3.5-27B local execution via ollama with tool simulation wrapper
- Gemini API integration with search grounding and thinking mode

### Out of Scope

- Modifying Claude Code itself — only hooks, agents, and plugin source code
- Running MiniMax locally (no open weights for M-2.7; API-only)
- Mobile or web interface — terminal/CLI only
- Auto-applying model changes without human approval — recommendations only
- Real-time streaming between models — async handoff is sufficient
- Supporting models not in the nine-model roster without explicit user request

## Context

- **Renamed from:** Claude X Codex (2026-04-04)
- **Design spec:** `docs/specs/2026-04-04-seraphim-v3-design.md`
- **Existing hooks:** 18 scripts at `~/.claude/hooks/` — 7 will be consolidated into plugin, rest kept
- **Dashboard:** `~/.claude/dashboard/dashboard.html` — will be extended with Seraphim metrics
- **Hardware:** RTX 3090 in transit for local Qwen 3.5-27B inference
- **Research basis:** `minimax-m2.7-synthesis.md`, `docs/research/opus-vs-codex-model-comparison.md`
- **Runtime:** Ubuntu 24.04, Claude Code CLI, Codex CLI v0.118.0, OpenAI SDK v6.33.0
- **Subscription:** $20/mo ChatGPT Plus, Perplexity paid account, Google AI Studio account needed
- **API keys configured:** OPENAI_API_KEY, MINIMAX_API_KEY. Needed: GEMINI_API_KEY
- **MCP configured:** Perplexity, context7, Brave, GitHub, Playwright, filesystem
- **SearXNG:** Running at localhost:8888

## Constraints

- **Budget**: $20/mo ChatGPT Plus; $15/day max API spend; local LLM preferred where quality allows
- **Security**: API keys in environment variables only; never send credentials or PII to MiniMax; bind services to 127.0.0.1
- **Hardware**: RTX 3090 required for Qwen 3.5-27B local inference; Balanced and Budget profiles unavailable without GPU
- **Orchestration**: Opus is the session host; it routes to phase models but doesn't own every phase
- **Plugin**: Must work as a standalone Claude Code plugin; no dependency on GSD or Superpowers at runtime
- **Fallback**: If a model is unavailable, the executor reports failure; the dispatch layer falls back per profile config

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| Hard fork from GSD | Too many customizations fighting against GSD's update cycle; own the workflow | v3.0 — clean break |
| Six phases from first principles | GSD's four phases are code-centric; six phases model the universal creative process | v3.0 — Discover, Envision, Judge, Architect, Forge, Crucible |
| Nine models, five profiles | Different cost/quality tradeoffs for different situations; profiles are presets, not locks | v3.0 — full override system |
| Unified executor interface | dispatch.js doesn't care which model; adding a new model = one file | v3.0 — clean abstraction |
| Qwen for local inference | Zero-cost execution on user hardware; 95% instruction following; strong coding benchmarks | v3.0 — requires RTX 3090 |
| Gemini 3 Flash for Judge | 90.4% GPQA Diamond (highest available); fast; cheap; adversarial thinking | v3.0 |
| Perplexity Sonar for external research | Purpose-built for web research; returns citations; better than any LLM at finding information | v3.0 |
| Adaptive intelligence via decision logging | Learn which models perform best per phase from real data, not benchmarks | v3.0 — recommendations require human approval |
| Consolidate hooks into pipeline | 7 hooks become redundant; one system to maintain; cleaner architecture | v3.0 |
| Rename project to Seraphim | Six wings of the seraph = six phases; mythological + cyber aesthetic; reflects evolved identity | v3.0 |
| Both CLI + API for Codex | CLI for autonomous execution (uses subscription), API for fast model-to-model comms | ✓ Good — v1.0 |
| Hook-based integration (v1-v2) | Non-invasive; survives plugin updates; uses native Claude Code hooks API | ✓ Good — v1.0, superseded by plugin in v3.0 |
| MiniMax via OpenAI SDK baseURL swap | Zero new deps; reuses existing openai v6.33.0 package | ✓ Good — v2.0 |
| MiniMax as adversarial reviewer | Different reasoning patterns (self-evolution trained) produce different critiques | ✓ Good — v2.0 |
| Fail-closed execution, fail-open review | Execution tasks prompt user on both-fail; review tasks skip silently | ✓ Good — v2.0 |

## Evolution

This document evolves at phase transitions and milestone boundaries.

**After each phase transition:**
1. Requirements invalidated? → Move to Out of Scope with reason
2. Requirements validated? → Move to Validated with phase reference
3. New requirements emerged? → Add to Active
4. Decisions to log? → Add to Key Decisions
5. "What This Is" still accurate? → Update if drifted

**After each milestone:**
1. Full review of all sections
2. Core Value check — still the right priority?
3. Audit Out of Scope — reasons still valid?
4. Update Context with current state

---
*Last updated: 2026-04-09 — v3.2 Idea-to-Shipped Journey milestone started*
