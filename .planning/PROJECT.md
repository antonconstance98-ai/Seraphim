# Claude X Codex

## What This Is

A three-model integration wiring OpenAI Codex (GPT-5.4) and MiniMax M-2.7 into an existing Claude Code + GSD + Superpowers workflow. Claude Opus 4.6 orchestrates and architects, Codex handles code execution (with MiniMax as fallback when Codex is rate-limited), and MiniMax provides adversarial review, bug scanning, and context compression — all coordinated through a cross-model plan review loop before any code is written. The system includes 12+ hook scripts, a Superpowers skill override, automated cost reporting, and a self-contained HTML dashboard showing global metrics across all projects on this machine.

## Core Value

Every task goes to the model that's best at it — Opus for reasoning and architecture, Codex for code execution, MiniMax for analysis and adversarial review — with cross-model review catching what any single model misses alone.

## Shipped: v2.0 Three-Model Intelligence (2026-04-03)

All v2.0 features delivered across 7 phases, 12 plans. MiniMax M-2.7 fully integrated as third model.

## Current State

**v2.0 shipped 2026-04-03.** 14 phases total across 3 milestones, 27 plans, 43 tasks.

Hook scripts installed at `~/.claude/hooks/` (18 total):
- `codex-exec.js` — Codex CLI wrapper with timeout, token parsing, GPT-5.4-mini API
- `codex-router.js` — PreToolUse advisory routing (opt-out, v2.0)
- `codex-token-logger.js` — PostToolUse token logging to JSONL (v2.0 field pass-through)
- `codex-review-gate.js` v3.0 — Stop hook parallel Codex+MiniMax dual review
- `codex-wave-validator.js` + `codex-wave-validator-worker.js` — Non-blocking GSD wave validation
- `codex-plan-reviewer.js` v3.1 — SubagentStop cross-model plan review
- `codex-multi-round-reviewer.js` v4.0 — Round 1 Codex constructive + Round 2 MiniMax adversarial
- `codex-superpowers-plan-reviewer.js` v3.1 — Superpowers SubagentStop cross-model review
- `codex-cost-reporter.js` — SessionStart three-model cost savings report
- `codex-pricing.js` — Centralized pricing (GPT-5.4 + Opus 4.6 + MiniMax M-2.7)
- `codex-global-aggregator.js` — SessionStart global JSONL aggregator across all projects
- `codex-dashboard-generator.js` — Three-model dashboard with Chart.js + Fallback Events panel
- `codex-handoff.js` — Execution helper: Codex CLI → MiniMax API → user prompt fallback chain
- `minimax-exec.js` v1.1 — MiniMax M-2.7 provider (runMinimax, runWithFallback, isCodexRateLimited)
- `minimax-post-scan.js` — PostToolUse bug/security scanner via MiniMax
- `minimax-compress.js` — Dual-mode compression (PostToolUse hook + require-able library)
- `minimax-connectivity-test.js` — MiniMax API connectivity verification
- `migrate-opus-pricing.js` — One-shot historical pricing correction (ran once)

Dashboard at `~/.claude/dashboard/dashboard.html` — three-model charts, auto-regenerated on every session start.

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

### Active

(None — next milestone requirements TBD via `/gsd:new-milestone`)

### Out of Scope

- Building a new CLI tool or standalone app — this integrates into existing Claude Code plugins
- Modifying Claude Code itself — only hooks, agents, and plugin source code
- Running MiniMax locally (no open weights for M-2.7; API-only, no filesystem access)
- Head-to-head benchmark suite — research already provides extensive data; track real-world A/B instead
- Real-time streaming between models — async handoff is sufficient
- Mobile or web interface — this is terminal/CLI only

## Context

- **Shipped:** v1.1 with 12 hook scripts, 1 Superpowers skill override, global metrics dashboard
- **Dashboard:** `~/.claude/dashboard/dashboard.html` — dark-themed, Chart.js charts, session drill-down, auto-regenerated on SessionStart
- **Research basis:** `minimax-m2.7-synthesis.md` — 8 research sources (3 Claude agents + 5 ChatGPT Deep Research reports), `minimax-m2.7-research.md` — raw Claude research, `research/` — 5 ChatGPT report source files
- **Key finding:** Cross-model review produces significantly better results than either model alone; MiniMax's self-evolution training gives it genuinely different reasoning patterns ideal for adversarial review
- **Cost efficiency:** Savings recalculated with corrected Opus 4.6 pricing ($5/$25). Old baseline ($173.74) was 3x overstated; new baseline ($57.91). Real savings now accurately reported.
- **Runtime:** Ubuntu 24.04, Claude Code CLI v2.1.91, Codex CLI v0.118.0, OpenAI SDK v6.33.0
- **Subscription:** $20/mo ChatGPT Plus (CLI usage preferred; subject to 5-hour rolling window and weekly caps)
- **MiniMax credits:** $25 available for M-2.7 API testing
- **MiniMax API:** OpenAI-compatible at `https://api.minimax.io/v1`, also Anthropic-compatible at `https://api.minimax.io/anthropic`, model `MiniMax-M2.7`, $0.30/$1.20 per M tokens, cache read $0.06/M
- **MiniMax architecture:** 230B total params, ~10B active (MoE, 23:1 sparsity), 204K context, 131K max output
- **MiniMax strengths:** SWE-Pro 56.22% (tied frontier), bug detection 6/6 and security scanning 10/10 (matched Opus in head-to-head), low hallucination (34%), 500 RPM / 20M TPM rate limits
- **MiniMax weaknesses:** No filesystem access (API-only, not a CLI tool), `<think>` tag fragility in multi-turn, no strict JSON mode, temperature cannot be exactly 0, verbosity tax (~4x output tokens), weaker on architecture and integration test depth
- **MiniMax quirks:** Must preserve full assistant messages including `<think>` content and tool_calls in conversation history; temperature must be (0.0, 1.0]; Codex CLI integration explicitly "not recommended" by MiniMax
- **Prior attempt:** ~/projects/Model-routing built a Fastify proxy — failed due to OAuth rejection, stream bugs, crashes. Lesson: integrate directly via SDK, not proxy
- **Execution gap (v1.x):** gsd-executor subagents run on Opus — all code writing burns Opus tokens. v2.0 moves code writing to Codex CLI (free) with MiniMax API fallback (cheap)
- **User is non-technical** — all changes implemented by Claude, explained in plain English

## Constraints

- **Budget**: $20/mo ChatGPT Plus subscription; $15/day max API spend; prefer CLI over API billing; MiniMax API as cost-efficient fallback
- **Security**: API keys in environment variables only; CLAUDE_CODE_SUBPROCESS_ENV_SCRUB=1 active; MiniMax data privacy policy not fully transparent — don't send credentials or PII
- **Compatibility**: Works with existing GSD and Superpowers plugin versions without breaking workflows
- **Runtime**: Codex CLI runs locally with filesystem access; MiniMax is API-only (no filesystem access, orchestrator writes files on its behalf)
- **Orchestration**: Opus always remains the primary orchestrator; neither Codex nor MiniMax make architectural decisions autonomously
- **Fallback chain**: Codex CLI (free) → MiniMax API ($0.30/$1.20) → prompt user. Execution is fail-closed (prompt user); reviews are fail-open (skip if both fail)

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
| MiniMax via OpenAI SDK baseURL swap | Zero new deps; reuses existing openai v6.33.0 package | ✓ Good — 3.5s connectivity, $0.000136 test call |
| Temperature 0.01 for MiniMax | API rejects exactly 0; 0.01 is the established workaround | ✓ Good — deterministic enough for review tasks |
| 90s timeout for MiniMax | Mandatory `<think>` phase can produce 55s pre-answer latency | ✓ Good — covers P95 latency without blocking |
| Direct fs.appendFileSync for token logging | Hooks write directly to JSONL; token-logger passes through CODEX_RESULT markers | ✓ Good — consistent pattern across all hooks |
| Dual review: parallel Promise.all | Independent .catch() wrappers per leg; either model can block independently | ✓ Good — graceful degradation if either fails |
| MiniMax as adversarial Round 2 (not Codex) | Different reasoning patterns (self-evolution trained) produce genuinely different critiques | ✓ Good — catches issues Codex misses |
| Codex handoff thin orchestrator | gsd-executor generates specs, Codex CLI writes code (free via subscription) | ✓ Good — expected cost drop from $10-15/day to $1-3/day |
| Fail-closed execution, fail-open review | Execution tasks prompt user on both-fail; review tasks skip silently | ✓ Good — matches risk profile of each task type |

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
*Last updated: 2026-04-03 after v2.0 milestone completion*
