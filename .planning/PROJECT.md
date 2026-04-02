# Claude X Codex

## What This Is

A multi-model integration that wires OpenAI Codex (GPT-5.4) into an existing Claude Code + GSD + Superpowers workflow. Claude Opus 4.6 acts as the orchestrator/architect while Codex handles fast execution and review — with a multi-round cross-model plan review loop before any code is written. The system includes 7 hook scripts, a Superpowers skill override, and automated cost reporting that proved 86.7% savings vs Opus-only in testing.

## Core Value

Every task goes to the model that's best at it — Opus for reasoning and architecture, Codex for fast execution — with cross-model review catching what either model misses alone.

## Current Milestone: v1.1 Global Metrics Dashboard

**Goal:** A single HTML dashboard at `~/.claude/dashboard/` showing Codex usage, costs, savings, and review activity across every project on this machine — auto-updated on every session start.

**Target features:**
- Global token log aggregator scanning all projects for `token-log.jsonl` files
- Per-project metrics breakdown (costs, savings, tokens, models, review catch rates)
- Time trend charts showing costs and savings over time (daily/weekly)
- Session history listing recent sessions per project with individual stats
- Self-contained HTML dashboard (inline CSS/JS, no server needed)
- SessionStart hook integration to auto-regenerate the dashboard

## Current State

**v1.0 shipped 2026-04-02.** All 4 phases complete (8 plans, 14 tasks, 8,694 lines added).

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

## Requirements

### Validated

- [x] Claude config hooks auto-route simple/defined tasks to Codex CLI — *v1.0*
- [x] Built-in token usage tracking logs every model call with tokens used, cost, and task type — *v1.0*
- [x] Opus remains the sole model for architectural decisions and complex reasoning tasks — *v1.0 (AGENTS.md hard-stop rule)*
- [x] GSD plugin modified to use Codex at specific workflow points — *v1.0*
- [x] Codex CLI preferred over API calls wherever possible — *v1.0*
- [x] Superpowers plugin modified to use Codex for parallel hypothesis testing — *v1.0*
- [x] Opus-Codex plan review loop (2 rounds) triggers before every plan — *v1.0*
- [x] OpenAI API used only for quick model-to-model communication — *v1.0*
- [x] Token tracking generates session reports showing savings vs Opus-only baseline — *v1.0*

### Active

- [ ] Global token log aggregator scans all projects for token-log.jsonl files
- [ ] Per-project metrics breakdown (costs, savings, tokens, models, review catch rates)
- [ ] Time trend charts showing costs and savings over time (daily/weekly)
- [ ] Session history listing recent sessions per project with individual stats
- [ ] Self-contained HTML dashboard at ~/.claude/dashboard/ (inline CSS/JS, no server)
- [ ] SessionStart hook auto-regenerates the dashboard on every session
- [ ] Opus generates adaptive handoff specs (file-level for complex, feature-level for simple) for Codex execution

### Out of Scope

- Building a new CLI tool or standalone app — this integrates into existing Claude Code plugins
- Modifying Claude Code itself — only hooks, agents, and plugin source code
- Supporting models beyond OpenAI's Codex family — no Gemini, no Llama
- Real-time streaming between models — async handoff is sufficient
- Mobile or web interface — this is terminal/CLI only

## Context

- **Shipped:** v1.0 with 7 hook scripts, 1 Superpowers skill override, 8,694 LOC (Node.js)
- **Research basis:** `docs/research/opus-vs-codex-model-comparison.md` — 34 sources informing routing decisions
- **Key finding:** Cross-model review produces significantly better results than either model alone
- **Cost efficiency:** 86.7% savings demonstrated in functional testing ($0.0233 actual vs $0.1759 Opus baseline)
- **Runtime:** Ubuntu 24.04, Claude Code CLI v2.1.90, Codex CLI v0.118.0, OpenAI SDK v6.33.0
- **Subscription:** $20/mo ChatGPT Plus (CLI usage preferred over API billing)
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
*Last updated: 2026-04-02 after v1.1 milestone start*
