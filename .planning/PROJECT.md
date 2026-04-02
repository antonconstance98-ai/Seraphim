# Claude X Codex

## What This Is

A multi-model integration that adds OpenAI Codex (GPT-5.4) capabilities into an existing Claude Code workflow. It modifies the Claude Code configuration (hooks, agents), the GSD (Get Shit Done) plugin, and the Superpowers plugin so that Claude Opus 4.6 acts as the orchestrator/architect while Codex handles implementation execution — with a cross-model plan review loop before any code is written. The goal is better results at lower cost by routing each task to the model that's best at it.

## Core Value

Every task goes to the model that's best at it — Opus for reasoning and architecture, Codex for fast execution — with cross-model review catching what either model misses alone.

## Requirements

### Validated

(None yet — ship to validate)

### Active

- [ ] Claude config hooks auto-route simple/defined tasks to Codex CLI
- [ ] GSD plugin modified to use Codex at specific workflow points (post-execution review, background validation, cross-wave integration checking)
- [ ] Superpowers plugin modified to use Codex for parallel hypothesis testing, parallel code reviews, and parallel verification
- [ ] Opus-Codex plan review loop (2-3 rounds) triggers before every phase plan and every individual task plan
- [ ] Opus generates adaptive handoff specs (file-level for complex, feature-level for simple) for Codex execution
- [ ] Codex CLI preferred over API calls wherever possible (maximizing $20/mo ChatGPT Plus subscription)
- [ ] OpenAI API used only for quick model-to-model communication where CLI overhead is impractical
- [ ] Built-in token usage tracking logs every model call with tokens used, cost, and task type
- [ ] Token tracking generates session reports showing savings vs Opus-only baseline
- [ ] Opus remains the sole model for architectural decisions and complex reasoning tasks

### Out of Scope

- Building a new CLI tool or standalone app — this integrates into existing Claude Code plugins
- Modifying Claude Code itself — only hooks, agents, and plugin source code
- Supporting models beyond OpenAI's Codex family — no Gemini, no Llama
- Real-time streaming between models — async handoff is sufficient
- Mobile or web interface — this is terminal/CLI only

## Context

- **Research basis:** `docs/research/opus-vs-codex-model-comparison.md` contains benchmark data from 34 sources informing all routing decisions
- **Key research finding:** Cross-model review (Opus reviews Codex output, Codex reviews Opus output) produces significantly better results than either model alone
- **Token efficiency:** Codex uses ~4x fewer tokens per equivalent task; combined with 2-6x cheaper per-token pricing, a $1.00 Opus task costs ~$0.10-0.15 on Codex
- **Plugin ecosystems:** GSD uses `.planning/` directory with phases, plans, and agent orchestration. Superpowers uses skills with parallel agent dispatch.
- **Runtime environment:** Ubuntu 24.04, Claude Code CLI, Codex CLI, OpenAI API access
- **Subscription:** $20/mo ChatGPT Plus (prefer CLI usage over API billing)
- **User is non-technical** — all changes must be implemented by Claude, explained in plain English

## Constraints

- **Budget**: $20/mo ChatGPT Plus subscription; $15/day max API spend; prefer CLI over API billing
- **Security**: Never expose API keys in plaintext; use environment variables; bind services to 127.0.0.1
- **Compatibility**: Must work with existing GSD and Superpowers plugin versions without breaking current workflows
- **Runtime**: Codex CLI runs locally in terminal; API calls use OpenAI SDK
- **Orchestration**: Opus always remains the primary orchestrator; Codex never makes architectural decisions autonomously

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| Both CLI + API for Codex | CLI for autonomous execution (uses subscription), API for fast model-to-model comms | — Pending |
| Modify plugin source directly | First-class integration, not bolted-on wrappers | — Pending |
| Adaptive handoff specs | File-level detail for complex/risky; feature-level for simple — avoids unnecessary overhead | — Pending |
| Review loop at phase AND plan level | Thorough cross-model review catches more issues; research confirms 2-3 rounds optimal | — Pending |
| Built-in token tracking | Required to prove cost savings — the success metric for "done" | — Pending |

## Evolution

This document evolves at phase transitions and milestone boundaries.

**After each phase transition** (via `/gsd:transition`):
1. Requirements invalidated? → Move to Out of Scope with reason
2. Requirements validated? → Move to Validated with phase reference
3. New requirements emerged? → Add to Active
4. Decisions to log? → Add to Key Decisions
5. "What This Is" still accurate? → Update if drifted

**After each milestone** (via `/gsd:complete-milestone`):
1. Full review of all sections
2. Core Value check — still the right priority?
3. Audit Out of Scope — reasons still valid?
4. Update Context with current state

---
*Last updated: 2026-04-02 after initialization*
