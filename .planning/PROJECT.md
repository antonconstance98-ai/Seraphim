# Claude X Codex

## What This Is

A multi-model integration that wires OpenAI Codex (GPT-5.4) into an existing Claude Code + GSD + Superpowers workflow. Claude Opus 4.6 acts as the orchestrator/architect while Codex handles fast execution and review — with a multi-round cross-model plan review loop before any code is written. The system includes 10 hook scripts, a Superpowers skill override, automated cost reporting, and a self-contained HTML dashboard showing global metrics across all projects on this machine.

## Core Value

Every task goes to the model that's best at it — Opus for reasoning and architecture, Codex for fast execution — with cross-model review catching what either model misses alone.

## Current Milestone: v2.0 Three-Model Intelligence

**Goal:** Integrate MiniMax M2.7 as a third independent model alongside Opus and GPT-5.4 — adding a genuinely different perspective to review workflows, specializing in debugging/test generation, and building a head-to-head benchmark suite to inform optimal task routing.

**Target features:**
- MiniMax M2.7 integration into existing hook infrastructure (direct SDK, no proxy)
- Third-perspective review pass in the cross-model review loop
- M2.7 as dedicated debugging and test generation specialist
- Head-to-head benchmark suite (M2.7 vs GPT-5.4) across multiple task types
- Benchmark results dashboard/reporting
- Three-model token tracking and cost reporting
- Updated global dashboard for three-model metrics

## Current State

**v1.1 shipped 2026-04-03.** 7 phases total (3 new in v1.1), 15 plans, 25 tasks.

Hook scripts installed at `~/.claude/hooks/`:
- `codex-exec.js` — Codex CLI wrapper with timeout, token parsing, GPT-5.4-mini API
- `codex-router.js` — PreToolUse advisory routing (opt-out, v2.0)
- `codex-token-logger.js` — PostToolUse token logging to JSONL
- `codex-review-gate.js` — Stop hook ALLOW/BLOCK review with depth variation
- `codex-wave-validator.js` + `codex-wave-validator-worker.js` — Non-blocking GSD wave validation
- `codex-plan-reviewer.js` — SubagentStop multi-round plan review (v3.0)
- `codex-multi-round-reviewer.js` — Shared 2-round review loop orchestrator
- `codex-superpowers-plan-reviewer.js` — Superpowers SubagentStop plan review
- `codex-cost-reporter.js` — SessionStart cost savings report
- `codex-pricing.js` — Centralized pricing module (GPT + Opus rates)
- `codex-global-aggregator.js` — SessionStart global JSONL aggregator across all projects
- `codex-dashboard-generator.js` — HTML dashboard generator with Chart.js visualizations

Dashboard at `~/.claude/dashboard/dashboard.html` — auto-regenerated on every session start.

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

### Active

- [ ] MiniMax M2.7 integrated into hook infrastructure via direct OpenAI SDK calls (no proxy)
- [ ] Third-perspective review pass added to cross-model review loop
- [ ] M2.7 serves as dedicated debugging and test generation specialist
- [ ] Head-to-head benchmark suite compares M2.7 vs GPT-5.4 across multiple task types
- [ ] Benchmark results reporting shows where each model excels
- [ ] Token tracking and cost reporting updated for three models
- [ ] Global dashboard updated to show three-model metrics
- [ ] Opus generates adaptive handoff specs (file-level for complex, feature-level for simple) for Codex execution

### Out of Scope

- Building a new CLI tool or standalone app — this integrates into existing Claude Code plugins
- Modifying Claude Code itself — only hooks, agents, and plugin source code
- Supporting models beyond OpenAI's Codex family — no Gemini, no Llama
- Real-time streaming between models — async handoff is sufficient
- Mobile or web interface — this is terminal/CLI only

## Context

- **Shipped:** v1.1 with 12 hook scripts, 1 Superpowers skill override, global metrics dashboard
- **Dashboard:** `~/.claude/dashboard/dashboard.html` — dark-themed, Chart.js charts, session drill-down, auto-regenerated on SessionStart
- **Research basis:** `docs/research/opus-vs-codex-model-comparison.md` — 34 sources informing routing decisions
- **Key finding:** Cross-model review produces significantly better results than either model alone
- **Cost efficiency:** 81.4% savings across 4 projects (global dashboard measurement)
- **Runtime:** Ubuntu 24.04, Claude Code CLI v2.1.90, Codex CLI v0.118.0, OpenAI SDK v6.33.0
- **Subscription:** $20/mo ChatGPT Plus (CLI usage preferred over API billing)
- **MiniMax credits:** $25 available for M2.7 API testing
- **MiniMax API:** OpenAI-compatible at `https://api.minimax.io/v1`, model `MiniMax-M2.7`, $0.30/$1.20 per M tokens
- **MiniMax strengths:** SWE-Pro 56.22% (within 1-2pts of Opus/GPT-5.4), AIME 2025 93.3% (beats both), low hallucination (34%), self-evolution training creates different reasoning patterns
- **MiniMax quirks:** Must preserve thinking content in history; stubborn about distributed rule files (rules must go in main prompt); less proactive about architecture/tests unless asked
- **Prior attempt:** ~/projects/Model-routing built a Fastify proxy — failed due to OAuth rejection, stream bugs, crashes. Lesson: integrate directly via SDK, not proxy
- **User is non-technical** — all changes implemented by Claude, explained in plain English

## Constraints

- **Budget**: $20/mo ChatGPT Plus subscription; $15/day max API spend; prefer CLI over API billing
- **Security**: API keys in environment variables only; CLAUDE_CODE_SUBPROCESS_ENV_SCRUB=1 active
- **Compatibility**: Works with existing GSD and Superpowers plugin versions without breaking workflows
- **Runtime**: Codex CLI runs locally in terminal; API calls use OpenAI SDK
- **Orchestration**: Opus always remains the primary orchestrator; Codex never makes architectural decisions autonomously

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| Both CLI + API for Codex | CLI for autonomous execution (uses subscription), API for fast model-to-model comms | ✓ Good — CLI handles all review/execution; API added for GPT-5.4-mini dispatch |
| Hook-based integration (not plugin source modification) | Non-invasive; survives plugin updates; uses native Claude Code hooks API | ✓ Good — 7 hooks, zero plugin source changes |
| Advisory routing (not auto-delegation) | Opus decides whether to delegate; safer than keyword-based auto-routing | ✓ Good — prevents cost runaway, Opus retains judgment |
| Multi-round review (constructive + adversarial) | Research confirms 2 distinct review types catch more issues than 2 identical passes | ✓ Good — early exit on clean plans saves tokens |
| Built-in token tracking + cost reporting | Required to prove cost savings — the success metric for "done" | ✓ Good — 86.7% savings demonstrated |
| User-space skill override for Superpowers | ~/.claude/skills/ shadows plugin cache, survives auto-updates | ✓ Good — durable without modifying plugin source |
| Centralized pricing module (codex-pricing.js) | Single source of truth for all model pricing; two functions for different safety levels | ✓ Good — eliminated duplicated constants, backward-compatible re-export chain |
| Configurable discovery roots with defaults | Hardcoded roots miss projects in unexpected locations; config.json extensibility | ✓ Good — covers all CLAUDE.md key paths, users can add more |
| Append-only global.jsonl with in-memory dedup | Preserves historical data even if per-project logs are deleted | ✓ Good — idempotent, no data loss risk |
| TTL-gated discovery cache (1hr) | Warm no-op runs under 5ms by skipping 4x spawnSync find processes | ✓ Good — 2ms warm runs vs 151ms cold |
| Chart.js sidecar with SHA-256 integrity | Downloaded once, inlined into HTML at gen time; works offline | ✓ Good — 204KB cached, integrity-verified |
| Atomic write-then-rename for dashboard.html | Prevents concurrent session corruption on shared output file | ✓ Good — Linux renameSync is atomic |

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
*Last updated: 2026-04-03 after v2.0 milestone start*
