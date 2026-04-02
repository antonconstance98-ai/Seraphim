# Project Research Summary

**Project:** Claude X Codex — Multi-Model AI Coding Agent Integration
**Domain:** Claude Code plugin modification — Opus 4.6 + Codex CLI orchestration
**Researched:** 2026-04-02
**Confidence:** HIGH

## Executive Summary

This project augments an already-working Claude Code + GSD + Superpowers setup by wiring in the OpenAI Codex CLI as a second model for code execution and review. The recommended approach is entirely additive: write Node.js hook scripts that call the existing `codex-companion.mjs` bridge, use filesystem-based handoff (`.planning/codex-handoff/`) for plan-to-Codex communication, and log token usage to `.planning/token-log.jsonl` after every Codex call. All prerequisites are already installed — Node.js v22.22.0, Codex CLI 0.118.0, GSD plugin, Superpowers plugin, and the official `codex-plugin-cc`. No new runtimes or frameworks are needed.

The recommended build order prioritises the token tracker and handoff spec format first (they are success-metric infrastructure and everything else depends on them), then adds the Stop hook review gate and GSD/Superpowers integration points, and finishes with the adaptive review loop and cost reporting. The central value proposition — routing execution tasks to Codex (fast, cheap) while Opus stays as architect — is validated by the HubSpot Sidekick case study (80% engineer approval, 90% faster PR feedback) and Chandler Nguyen's practitioner research confirming 2-3 review rounds produce measurably better output than either model alone.

The critical risks are: a known Codex CLI background-process hang bug (must wrap every `codex exec` call with `timeout 300`), API key exfiltration through hook scripts (CVE patched in Claude Code 2.0.65+, must audit hooks before wiring keys), cost runaway if routing logic fails open to Opus, and ChatGPT Plus quota exhaustion (Codex-Spark is Pro-only — replace all Codex-Spark routing with `gpt-5.4-mini` via API). All four risks have specific, implementable mitigations that belong in Phase 1 before any production routing is enabled.

---

## Key Findings

### Recommended Stack

The entire integration layer runs in Node.js (v22.22.0, already installed) using four components that are already present on the system. The OpenAI SDK (`openai@6.33.0`) is the only new install needed, and only for API-based fallback calls when CLI quota runs low. All hook scripts follow the existing GSD/Superpowers pattern: read JSON from stdin, write JSON to stdout, exit 0 for success, exit 2 to block.

**Core technologies:**
- **Codex CLI 0.118.0** (`~/.npm-global/bin/codex`): non-interactive execution via `codex exec --json`; uses ChatGPT Plus subscription (free under $20/mo cap) — preferred over API for long-running tasks
- **`codex-companion.mjs`** (official Codex plugin bridge): job state management, background/foreground execution, thread persistence — always route through this, never call `codex` CLI directly
- **Claude Code hooks** (`PostToolUse`, `Stop`, `PreToolUse`): the integration plumbing; all live in `~/.claude/settings.json` under the `hooks` key
- **OpenAI Node.js SDK 6.33.0**: direct API calls (`gpt-5.4-mini`) for short, structured advisory queries where CLI startup overhead (~1-2s) is impractical
- **JSONL token log** (`.planning/token-log.jsonl`): append-only, written after every Codex call; provides the session cost data needed to prove savings vs Opus-only baseline

**What NOT to use:** OpenAI Agents SDK, LangChain, Claude Flow, `opencode` CLI, GPT-5.4 Pro ($30/$180/Mtok), Codex Cloud (Pro-only), or `async: true` hooks for review (async hooks cannot inject `additionalContext` that blocks Claude).

### Expected Features

**Must have (table stakes):**
- **AGENTS.md at repo root** — without it, Codex has no project context and hallucinates conventions
- **Codex CLI invocation from hooks** — the plumbing everything else depends on
- **Token usage logging per call** — required to prove the value proposition from day one
- **Stop hook review gate** — single highest-impact quality feature; the official plugin ships this but our version integrates with GSD phase gates
- **Cross-model code review** — primary reason to have two models; Claude implements, Codex reviews (or vice versa)
- **Fallback routing on Codex failure** — operational resilience; fail-closed to human prompt, not open to Opus
- **Explicit Opus-stays-architect boundary** — encoded in hooks, CLAUDE.md, and AGENTS.md; must be a hard rule

**Should have (differentiators over just installing codex-plugin-cc):**
- **Opus-Codex plan review loop (2-3 rounds)** — quality multiplier; research confirms single-pass review leaves significant issues
- **GSD wave-level Codex injection** — Codex background validation between waves before completion is confirmed
- **Session cost report vs Opus-only baseline** — the "proof it works" feature the user needs to trust the integration
- **Adaptive handoff spec generation** — file-level detail for risky/complex tasks; feature-level for routine ones
- **Background Codex validation** — non-blocking quality assurance; result available when Opus finishes

**Defer (v2+):**
- **Superpowers parallel hypothesis testing via Codex-Spark** — requires ChatGPT Pro ($200/mo); user has Plus ($20/mo); substitute with `gpt-5.4-mini` via API if hypothesis testing is needed
- **Rate-limit aware automatic routing** — implement only after hitting Plus limits in practice
- **Web UI or cost dashboard** — CLI report to `.planning/session-reports/` is sufficient for terminal-first workflow

### Architecture Approach

The integration uses three communication patterns across six components. Claude Code (Opus 4.6) is the permanent orchestrator: it generates plans, reads results, makes all architectural decisions, and coordinates subagents via GSD's `.planning/` filesystem state. Hook scripts (Node.js processes) intercept lifecycle events and trigger Codex via `codex-companion.mjs` — the bridge that manages job state, background workers, and thread persistence. Model-to-model communication uses file-based handoff: Opus writes `.planning/codex-handoff/{id}.md`, Codex reads it, writes response to `.planning/codex-responses/{id}.md`, Opus reads the result. No inline prompt stuffing; no streaming between models.

**Major components:**
1. **Claude Code session (Opus 4.6)** — primary orchestrator; all reasoning, planning, architectural decisions; reads/writes `.planning/` state
2. **Hook scripts** — Node.js processes that intercept `PostToolUse`/`Stop`/`PreToolUse` events; trigger Codex review; inject `additionalContext`; never synchronous in `PostToolUse` (background only)
3. **GSD plugin** — wave-based phase execution; integration points at plan-write, wave-boundary, and verification events
4. **Superpowers plugin** — behavioral skills; parallel agent dispatch; model routing hints added to `dispatching-parallel-agents` skill
5. **Codex companion runtime** (`codex-companion.mjs`) — bridge to Codex App Server; manages foreground/background jobs, status polling, result retrieval; always use this, never raw `codex` CLI
6. **Token tracker** (`.planning/token-log.jsonl`) — append-only JSONL; records model, task type, tokens (separated by type per provider), estimated cost, session ID after every call

### Critical Pitfalls

1. **Codex CLI silent background hang (Bug, CLI 0.111-0.114+)** — wrap every `codex exec` call with `timeout 300`; use `--approval-policy untrusted` to prevent persistent background subprocesses; never route long-running background-script tasks to Codex
2. **API key exfiltration via hook scripts (CVE-2025-59536)** — update Claude Code to 2.0.65+; enable `CLAUDE_CODE_SUBPROCESS_ENV_SCRUB=1`; never echo or log environment variables in hook scripts; audit all hook source files as security-sensitive before enabling
3. **Cost runaway from routing failure** — implement hard $10/day spend cap before Phase 2 parallel routing; make routing fail-closed (prompt human) not fail-open (route to Opus); cap Codex retries at 3 before abandoning, not falling back
4. **GSD plugin commands break after Claude Code updates** — pin GSD to a version tag before modifying source; maintain a patch file of all modifications; prefer extension points (hooks, skills) over direct source edits
5. **ChatGPT Plus quota exhaustion + Codex-Spark unavailable** — Codex-Spark is Pro-only ($200/mo); replace all routing rules referencing it with `gpt-5.4-mini` via API; track CLI quota separately from API spend; reserve Plus CLI for high-value, longer-running tasks

---

## Implications for Roadmap

Based on the dependency graph in FEATURES.md and the build order in ARCHITECTURE.md, five phases are suggested. Token tracking comes first because it is the success metric for the entire project — without it, there is no way to validate any subsequent phase. Security must be settled in Phase 1 before any API keys are wired into hooks.

### Phase 1: Foundation — Security, Plumbing, and Metrics

**Rationale:** AGENTS.md, headless auth, CVE remediation, timeout wrappers, token log schema, and handoff directory structure are all Phase 1 work per PITFALLS.md. Every subsequent phase builds on these. Cost guard rails must exist before any routing logic is written.
**Delivers:** Working `codex exec` call from a hook; AGENTS.md providing project context; token log capturing every Codex call; hard daily spend cap ($10 ceiling); handoff spec template stable
**Addresses (from FEATURES.md):** AGENTS.md spec, Codex CLI invocation from hooks, token usage logging, fallback routing on failure
**Avoids:** Pitfall 1 (timeout wrapper), Pitfall 2 (CVE + env scrubbing), Pitfall 4 (spend cap), Pitfall 8 (Spark routing replaced), Pitfall 10 (headless auth verified)
**Research flag:** Standard patterns — GSD hook structure is well-documented and live examples exist

### Phase 2: Core Review Gate and GSD Integration

**Rationale:** With plumbing and metrics stable, the Stop hook review gate is the single highest-impact feature. GSD integration points (plan-write trigger, wave-boundary check) are the primary use case for the integration.
**Delivers:** Stop hook that calls Codex synchronously, can block Claude from stopping, follows existing `stop-review-gate-hook.mjs` ALLOW/BLOCK protocol; PostToolUse hook that triggers Codex review on PLAN.md writes; cross-wave integration check between GSD waves
**Addresses (from FEATURES.md):** Stop hook review gate, cross-model code review, GSD wave-level Codex injection
**Avoids:** Pitfall 3 (GSD commands patched and smoke-tested), Pitfall 6 (token tracking accuracy validated against dashboards), Pitfall 7 (bounded tasks only — no open-ended Codex sessions yet), Pitfall 9 (Codex review calls staggered from Opus planning calls)
**Research flag:** May need brief research on GSD wave state schema before implementing wave-boundary hook

### Phase 3: Opus-Codex Plan Review Loop

**Rationale:** The review loop synthesises Phase 1 (handoff spec, token tracking) and Phase 2 (review gate, GSD integration). It is the "quality multiplier" that makes this better than single-pass review. Research confirms 2-3 rounds produce measurably better output.
**Delivers:** 2-3 round plan review workflow; round counter persisted in `.planning/review-state.json`; hard 3-round cap with Opus final authority; structured review prompts (elicit specific objections, not open-ended critique)
**Addresses (from FEATURES.md):** Opus-Codex plan review loop, adaptive handoff spec generation
**Avoids:** Pitfall 5 (typed handoff spec with `decisions_not_taken` section), Pitfall 12 (3-round cap enforced)
**Research flag:** Standard patterns — existing `stop-review-gate-hook.mjs` parseStopReviewOutput() pattern is the template

### Phase 4: Superpowers Integration and Background Validation

**Rationale:** Superpowers skills are more fragile than GSD workflows (skills-directory migration risk) and depend on Phase 2 patterns being established. Background validation requires the PostToolUse hook pattern from Phase 2.
**Delivers:** `dispatching-parallel-agents` skill updated with Codex model routing hints; background Codex validation during wave execution (non-blocking); Codex reads `.planning/validation-queue/` at natural stopping points
**Addresses (from FEATURES.md):** Background Codex validation during Claude execution, Superpowers parallel agent model routing
**Avoids:** Pitfall 2 (anti-pattern: synchronous Codex in every PostToolUse hook), Pitfall 7 (context degradation — background validation tasks are bounded)
**Research flag:** May need to verify `~/.agents/skills/` symlink path for Superpowers + Codex skill loading before implementing

### Phase 5: Cost Reporting and Validation

**Rationale:** Session cost reports require stable, accurate token tracking from all previous phases. The comparative report ("this session cost $X; Opus-only equivalent would have cost $Y") is the proof-of-value the user needs, and it cannot be accurate until the underlying tracking is validated.
**Delivers:** Session report generator reading `.planning/token-log.jsonl`; cost comparison against Opus-only baseline; report written to `.planning/session-reports/YYYY-MM-DD.md`; cross-validated against Anthropic and OpenAI billing dashboards for first two weeks
**Addresses (from FEATURES.md):** Session cost report vs Opus-only baseline, rate-limit aware routing (if Plus limits have been hit in practice by this phase)
**Avoids:** Pitfall 6 (final accuracy validation against provider dashboards)
**Research flag:** No research needed — token schema and pricing are verified; this is implementation work

### Phase Ordering Rationale

- Token tracking and security must precede any routing logic — they are prerequisites, not features
- GSD integration (Phase 2) precedes Superpowers (Phase 4) because GSD has clearer filesystem extension points; Superpowers skills are more fragile and depend on established patterns
- The review loop (Phase 3) requires both the handoff spec format (Phase 1) and the GSD integration points (Phase 2) to be stable before it is valuable
- Cost reporting (Phase 5) cannot be accurate until all token tracking sources are in production and validated

### Research Flags

Phases needing `/gsd:research-phase` during planning:
- **Phase 2:** GSD wave state schema — confirm the exact field names and file paths used in `.planning/STATE.md` for wave completion events before writing the wave-boundary hook
- **Phase 4:** Superpowers `~/.agents/skills/` symlink and Codex CLI skill loading path — verify this still works with Codex CLI 0.118.0 before modifying skill files

Phases with standard patterns (skip research):
- **Phase 1:** Hook protocol and Codex CLI invocation are fully documented with live examples
- **Phase 3:** `stop-review-gate-hook.mjs` is a working reference implementation for the ALLOW/BLOCK protocol
- **Phase 5:** Token schema verified; OpenAI and Anthropic pricing documented

---

## Confidence Assessment

| Area | Confidence | Notes |
|------|------------|-------|
| Stack | HIGH | Verified from live system (`node --version`, `codex --version`), official docs, and live JSONL schema inspection |
| Features | HIGH | Anchored by official docs (hooks, Codex CLI), official plugin source (codex-plugin-cc), and published case study (HubSpot Sidekick) |
| Architecture | HIGH | Based on live source code inspection of `codex-companion.mjs`, `stop-review-gate-hook.mjs`, GSD `execute-phase.md`, and Superpowers skill SKILL.md |
| Pitfalls | HIGH | Backed by GitHub issues, two CVE disclosures, and community-verified rate limit and quota reports |

**Overall confidence:** HIGH

### Gaps to Address

- **Adaptive handoff spec complexity classifier:** The distinction between "file-level detail" and "feature-level detail" for handoff specs is a design decision not yet validated. Treat the initial spec format as a v1 to be refined after first 10 real tasks.
- **Review loop round-trip timing:** Known that Codex executes at 240+ tok/s, but total wall-clock time for a 2-3 round review loop depends on plan size. Measure this in Phase 3 and cap at 5 minutes per round to avoid Stop hook timeout issues (15-minute limit).
- **Codex-Spark substitution quality:** Routing to `gpt-5.4-mini` instead of Codex-Spark for background validation is a workaround. Validate that `gpt-5.4-mini` quality is sufficient for background validation tasks before committing to it as the permanent approach.
- **GSD wave state schema:** The exact field names in `.planning/STATE.md` for wave completion events were not verified from source during research. Must be confirmed before writing the wave-boundary hook in Phase 2.

---

## Sources

### Primary (HIGH confidence)
- Claude Code Hooks API: https://code.claude.com/docs/en/hooks (verified 2026-04-02)
- Codex CLI `--help` output: verified on installed v0.118.0
- Live codebase: `codex-companion.mjs`, `stop-review-gate-hook.mjs`, `app-server.mjs`
- Live codebase: `gsd-context-monitor.js`, `execute-phase.md`, Superpowers `dispatching-parallel-agents/SKILL.md`
- Live session JSONL schema: verified from `~/.claude/projects/` files
- codex-plugin-cc: https://github.com/openai/codex-plugin-cc
- OpenAI Node.js SDK: https://github.com/openai/openai-node (v6.33.0 verified via `npm show openai`)
- HubSpot Sidekick multi-model code review: https://www.infoq.com/news/2026/03/hubspot-ai-code-review-agent/
- CVE-2025-59536 (API key exfiltration): https://research.checkpoint.com/2026/rce-and-api-token-exfiltration-through-claude-code-project-files-cve-2025-59536/

### Secondary (MEDIUM confidence)
- Chandler Nguyen — Dual-wielding Claude Code + Codex GPT-5.4: https://chandlernguyen.com/blog/2026/03/13/codex-gpt-5-4-vs-claude-code-opus-4-6-dual-wielding-ai-coding-tools/
- SmartScope codex-plugin-cc analysis: https://smartscope.blog/en/blog/codex-plugin-cc-openai-claude-code-2026/
- Hacker News GPT-5.4 discussion (cross-model review patterns): https://news.ycombinator.com/item?id=47265045
- Agentic Coding 2026 guide: https://halallens.no/en/blog/agentic-coding-in-2026-the-complete-guide-to-plugins-multi-model-orchestration-and-ai-agent-teams
- GitHub Issue #14303/#14314/#13708 — Codex CLI background hang: https://github.com/openai/codex/issues/14303
- GitHub Issue #13186 — Codex metering anomaly: https://github.com/openai/codex/issues/13186

### Tertiary (context / supporting)
- `docs/research/opus-vs-codex-model-comparison.md` — internal 34-source model comparison
- `.planning/PROJECT.md` — canonical project requirements and scope boundaries

---
*Research completed: 2026-04-02*
*Ready for roadmap: yes*
