---
gsd_state_version: 1.0
milestone: v3.0
milestone_name: Seraphim
status: executing
stopped_at: Completed 03-04-PLAN.md
last_updated: "2026-04-08T21:50:32.138Z"
last_activity: 2026-04-08
progress:
  total_phases: 15
  completed_phases: 6
  total_plans: 21
  completed_plans: 20
  percent: 0
---

# Project State

## Project Reference

See: .planning/PROJECT.md (updated 2026-04-04)

**Core value:** Six wings, six phases, six cognitive tasks — each assigned to the model that does it best. The human orchestrates. AI converges. Adaptive intelligence makes the system smarter over time.
**Current focus:** Phase 03 — six-phase-pipeline-and-profile-management

## Current Position

Phase: 03 (six-phase-pipeline-and-profile-management) — EXECUTING
Plan: 5 of 6
Status: Ready to execute
Last activity: 2026-04-08

Progress: [░░░░░░░░░░] 0%

## Performance Metrics

**Velocity (v1.0–v2.0 history):**

- Total plans completed: 27 (across previous milestones)
- Average duration: ~5 min/plan
- Total execution time: ~135 min

**By Phase:** — (v3.0 not started)

**Recent Trend:**

- Last 5 plans (v2.0): 2, 2, 3, 2, 6 min
- Trend: Stable

*Updated after each plan completion*

## Accumulated Context

### Decisions

Decisions are logged in PROJECT.md Key Decisions table.
Key decisions relevant to Phase 1:

- [v3.0 Design]: plugin.json must be at `.claude-plugin/plugin.json` — wrong path = silent failure (research confirmed)
- [v3.0 Design]: Use only `hooks/hooks.json` for hook declarations — never `plugin.json` to avoid double-registration silent failure
- [v3.0 Design]: `phase-state.js` persists loop counters to disk at every increment — in-memory counters lost on crash
- [v3.0 Design]: dispatch.js resolution order: override > opus_enabled flag > profile preset
- [Phase 01-plugin-scaffold-and-infrastructure]: plugin.json at .claude-plugin/plugin.json (not root) — wrong path causes silent failure where /seraphim: commands never register
- [Phase 01-plugin-scaffold-and-infrastructure]: No hooks key in plugin.json — hooks auto-discovered from hooks/hooks.json; declaring in both causes conflicting manifests error
- [Phase 01-plugin-scaffold-and-infrastructure]: Plugin git repo initialized at ~/.claude/plugins/seraphim/ (separate from project repo) to track plugin source files
- [Phase 01]: dispatch.js resolution order locked: override > opus_enabled > profile preset
- [Phase 01]: All dispatch error cases return {error: string} objects — callers check typeof result === 'string'
- [Phase 01]: phase-state.js writes synchronously on every mutation for crash safety over performance
- [Phase 01]: hooks.json auto-discovery prevents double-registration silent failure — never declare hooks in plugin.json
- [Phase 01]: session-start.js uses setTimeout.unref() so the 10s guard timer does not block normal process exit
- [Phase 01]: new-project.md references ${CLAUDE_PLUGIN_ROOT}/config/profiles.json and models.json for runtime profile data
- [Phase 02-model-executors-and-pricing]: fetchdocs.js uses claude CLI subprocess as MCP bridge — no confirmed public Context7 REST API; websearch.sh fallback ensures Codex/Qwen retain research capability
- [Phase 02-model-executors-and-pricing]: cache_read tokens are a positive charge at reduced rate (not a credit) — mishandling causes negative cost delta (COST-01)
- [Phase 02-model-executors-and-pricing]: token-logger.js writes to .seraphim/token-log.jsonl (not .planning/) per Seraphim per-project state convention
- [Phase 02-model-executors-and-pricing]: @google/genai@1.48.0 used (not deprecated @google/generative-ai); stateless GoogleGenAI client per call; { googleSearch: {} } grounding pattern (not google_search_retrieval); stream() delegates to execute() per FUTR-04
- [Phase 02-model-executors-and-pricing]: available() uses inference probe not /api/tags — forces VRAM load, catches cold-start GPU failures
- [Phase 02-model-executors-and-pricing]: Perplexity baseURL has no /v1 suffix — api.perplexity.ai routes /chat/completions directly off base
- [Phase 02-model-executors-and-pricing]: MCP path returns mcpRequest object to caller — MCP tools inaccessible from standalone Node.js
- [Phase 02-model-executors-and-pricing]: Both executors delegate fallback to dispatch.js — no cross-executor dependencies; runWithFallback removed from minimax fork
- [Phase 03-01]: EXECUTOR_MAP static lookup in dispatch.js CLI rather than models.json executorFile field — keeps mapping colocated with CLI code, avoids schema change
- [Phase 03]: Forge does NOT auto-commit (Pitfall 7) — Phase 4 checkpoint owns the commit gate
- [Phase 03]: Crucible adversarial pass always dispatched externally — MiniMax is never inline-Opus
- [Phase 03]: loop_required=true only when ALL approaches receive FATAL_FLAW — conditionals count as viable paths
- [Phase 03]: Architect selects SURVIVES first, CONDITIONAL fallback — CONDITIONAL selection is noted in blueprint overview
- [Phase 03]: TASK markers include type attribute (code/prose/analysis) even for homogeneous projects to enable Forge per-task branching

### Roadmap Evolution

- Phase 8 added: Thought Orphanage Integration (slash command for seed thought capture + dashboard representation)
- Phase 9 added: Human-AI Cognitive Division (research where human vs AI leverage sits in the pipeline)
- Phase 10 added: Context Management and Token Optimization (reduce token usage across nine-model pipeline)
- Phase 11 added: OpenClaw Local RAG Integration (local RAG for project knowledge referencing)

### Pending Todos

None yet.

### Blockers/Concerns

- [Phase 2]: Gemini SDK search grounding + thinking mode APIs need research verification before planning. Perplexity MCP bridge from Node.js executor needs design work — mechanism is not standard.
- [Phase 2]: RTX 3090 in transit — Qwen Balanced/Budget profiles unavailable until GPU arrives. Executor must exist and fail gracefully before GPU is installed.
- [Phase 4]: Cost-gate design before loop iterations needs implementation design (research flag).
- [Phase 5]: Non-code checkpoint design needs research — what a "research" or "writing" checkpoint actually verifies.
- [Coverage note]: REQUIREMENTS.md header states 52 requirements; actual count is 58 (7+9+11+5+6+5+6+6+3). Traceability table reflects actual 58.

## Session Continuity

Last session: 2026-04-08T21:50:23.716Z
Stopped at: Completed 03-04-PLAN.md
Resume file: None
