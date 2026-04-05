# Project Research Summary

**Project:** Seraphim v3.0
**Domain:** Standalone Claude Code plugin — six-phase multi-model creative pipeline
**Researched:** 2026-04-04
**Confidence:** HIGH

## Executive Summary

Seraphim v3.0 is a standalone Claude Code plugin that routes creative and engineering work through six cognitive phases (Discover, Envision, Judge, Architect, Forge, Crucible), each assigned to the model best suited to that task from a roster of nine models across five cost profiles. The project is a hard fork of GSD with no runtime dependency on GSD or Superpowers, built on top of two proven v2.0 components (codex-exec.js, minimax-exec.js) that are forked and extended rather than rebuilt. The design is well-specified and the domain is well-documented; the primary challenge is integration breadth (nine models, five external APIs, one local GPU inference stack) rather than architectural novelty.

The recommended build sequence is: scaffold the plugin structure and infrastructure first, wire up all nine model executors next, build the six-phase pipeline end-to-end, then add quality gates and feedback loops, and defer adaptive intelligence dashboard panels until the pipeline has accumulated enough real-world data to make recommendations meaningful. This ordering is driven by a hard dependency chain: executors must exist before dispatch.js can route, structured phase output schemas must be designed before feedback loops can parse them, and decisions.jsonl must be logging from the first pipeline run to give the analysis layer anything to work with.

The most consequential risks are: (1) hook double-registration silently disabling all plugin hooks, (2) Qwen cold-start timeouts silently routing to paid cloud fallbacks, (3) feedback loop counters stored in memory and lost across interrupted sessions, and (4) per-provider token cost formulas sharing a single implementation, producing negative or zero costs for Anthropic and Qwen calls respectively. All four are well-understood and preventable with explicit exit criteria at the phases that introduce them.

---

## Key Findings

### Recommended Stack

The v3.0 stack is almost entirely Node.js v22.22.0 with no new npm packages required beyond what v2.0 established. The two new SDK dependencies are `@google/generative-ai` for Gemini integration and an ollama HTTP client (no npm package — raw `fetch()` to localhost:11434). The ML/adaptive intelligence layer (if built in v3.0) adds `better-sqlite3@12.8.0`, `simple-statistics@7.8.9`, and `ml-regression@6.3.0`. Dashboard panels extend the existing Chart.js 4.5.1 HTML generator pattern. No build steps, no server processes, no web frameworks.

**Core technologies:**
- **Node.js v22.22.0:** All executor scripts, dispatch routing, hook scripts — synchronous API matches hook execution model
- **`@google/generative-ai` SDK:** Gemini 3.1 Pro (Discover internal, Envision Moderate) and Gemini 3 Flash (Judge) — search grounding, thinking mode, function calling all required
- **`openai` npm v6.33.0 (existing):** MiniMax API via baseURL swap; already installed globally at `~/.npm-global/`
- **ollama HTTP at localhost:11434:** Qwen 3.5-27B Q4 for Balanced/Budget profiles; RTX 3090 required (in transit); graceful degradation required
- **Chart.js 4.5.1 UMD (inlined):** Dashboard extension for heatmap and cost/quality panels; same integration pattern as v1.1
- **`better-sqlite3@12.8.0`:** Structured decision log for adaptive intelligence queries (Phase 8 only)
- **`simple-statistics@7.8.9`:** Statistical pattern analysis on decisions.jsonl — zero dependencies, pure JS

**Critical version notes:**
- `fs.glob()` on Node.js v22 returns an AsyncIterator, not a Promise. Use `for await...of`, never `.then()`.
- MiniMax rejects `temperature: 0` — use `0.01`.
- Qwen context must be capped at 32K by explicitly passing `num_ctx: 32768` in every ollama call; ollama default may be higher.

### Expected Features

The feature research distinguishes three delivery waves with a clear P1/P2/P3 priority stack.

**Must have — Plugin Core (Wave 1, P1):**
- `plugin.json` manifest at `.claude-plugin/plugin.json` (wrong path = silent failure)
- `dispatch.js` central router with three-level resolution: override > opus_enabled flag > profile preset
- `profiles.json` defining all five profiles with ten sub-phase slots each
- All nine model executors with uniform `execute()`, `stream()`, `available()` interface
- Six phase commands (`/seraphim:discover` through `/seraphim:crucible`) writing outputs to fixed disk paths
- Per-project config at `.seraphim/config.json` (profile, opus_enabled, overrides, max_loops)
- Token logging extended to all nine models with per-provider cost functions
- `decisions.jsonl` schema with outcome signals logged from the first pipeline run
- Helper scripts: `websearch.sh` (SearXNG) and `fetchdocs.js` (Context7) for non-MCP models
- `phase-state.js` with persisted loop counters and human escalation on cap exceeded

**Must have — Quality Gates (Wave 2, P1-P2):**
- Between-task checkpoints in Forge: runtime tests + lint + static model review
- Retry-with-feedback in Forge (max 2 retries per task, hard cap)
- Judge->Envision feedback loop with structured machine-readable output schema and hard cap
- Crucible->Forge feedback loop (verification + adversarial) with hard cap and cost-gate warning
- `/seraphim:run` full-auto mode with `--from [phase]` resume flag
- Non-code project type branching in Forge and Crucible (`project_type` field in blueprint.md)

**Should have — Adaptive Intelligence (Wave 3, P2-P3):**
- Pattern analysis engine (requires ~20+ pipeline runs to produce meaningful recommendations)
- Auto-recommendation system with human-approval guardrail (never auto-apply)
- Per-phase model performance heatmap panel in dashboard
- Profile cost/quality comparison panel in dashboard
- Hooks consolidation — retire seven redundant v2.0 hooks atomically once pipeline is verified

**Defer to v4 or later:**
- Cost estimate before run (pre-run token budget projection)
- Configurable loop cap per project (default 2 is sufficient for v3.0)
- GUI or web dashboard for pipeline control (terminal-only per PROJECT.md constraint)
- Supporting models outside the nine-model roster

### Architecture Approach

The plugin follows a layered architecture with clear boundary rules. Slash commands (`commands/*.md`) are Markdown prompt files that instruct the Opus host session what to do — they are not scripts. Phases where Opus is the primary model (Envision, Architect) run as native subagents. Phases where an external model is primary (Judge with Gemini Flash, Forge with Codex CLI) use an orchestrating agent that calls `dispatch.js` via Bash. `dispatch.js` is the only component that knows which executor to load; it is the sole routing abstraction. All executors implement exactly the same three-function interface. All mutable per-project state lives in `<project>/.seraphim/` — never in the plugin directory, which is wiped on plugin updates.

**Major components:**
1. **`plugin.json` + `commands/*.md`** — Plugin manifest and user-facing slash commands; Markdown prompt files parsed by Opus
2. **`dispatch.js` + `profiles.json` + `models.json`** — Central routing layer; resolves profile, overrides, and fallback chain; the single model-routing abstraction
3. **Five executor files** (`codex-exec.js`, `minimax-exec.js`, `gemini-exec.js`, `qwen-exec.js`, `perplexity-exec.js`) — Model-specific integration code; codex and minimax are forks from v2.0; the other three are new builds
4. **`lib/` support layer** (`pricing.js`, `config.js`, `phase-state.js`, `token-logger.js`) — Shared utilities; no external model calls; pricing.js must expose per-provider cost functions, never a generic formula
5. **`tools/`** (`websearch.sh`, `fetchdocs.js`, `checkpoint.js`) — Bridge helpers for non-MCP-capable models and Forge quality gates
6. **`hooks/`** (`session-start.js`, `token-logger.js`) — Plugin-owned hooks replacing the seven retiring global hooks; registered atomically in `~/.claude/settings.json` during plugin setup

### Critical Pitfalls

1. **Plugin manifest in wrong directory** — `plugin.json` must be at `.claude-plugin/plugin.json`, not at the plugin root. Claude Code silently ignores commands requiring manifest metadata. Prevention: scaffold creates `.claude-plugin/` in the first commit; `claude plugin validate` is the exit criterion.

2. **Hook double-registration** — Declaring hooks in both `plugin.json` and `hooks/hooks.json` triggers a `conflicting manifests` error that silently disables all plugin hooks. Prevention: use only `hooks/hooks.json`; never declare hooks in `plugin.json`.

3. **Old redundant hooks firing alongside new pipeline** — Seven v2.0 hooks remain active in `~/.claude/settings.json` and double-trigger every event, doubling costs and causing JSONL write conflicts. Prevention: disable old hooks at the START of the phase implementing their replacements, not at the end.

4. **Qwen cold-start timeout routing silently to paid cloud** — The `available()` check passes (ollama running) but the first inference request times out during model load (13-46s on RTX 3090). Prevention: 180s HTTP timeout in `qwen-exec.js`; warm-up probe before first real call; `available()` must send a minimal inference request, not just check `/api/tags`.

5. **Feedback loop counters lost across interrupted sessions** — In-memory counters reset on crash, allowing indefinite loops across restarts. Prevention: `phase-state.js` persists all loop counts to `.seraphim/phases/{N}/state.json` at every increment; persisted counts read at phase start.

6. **Token cost formula shared across providers** — Anthropic cache-read credits, Qwen's `prompt_eval_count`/`eval_count` fields, and MiniMax's cached-token semantics each require different handling. A shared formula produces negative costs for Anthropic and zero costs for Qwen. Prevention: `pricing.js` exposes per-provider cost functions only.

7. **Forked executors retaining old absolute path references** — `codex-exec.js` and `minimax-exec.js` contain `require('../codex-pricing')` paths that resolve to the old hook directory after forking. Prevention: replace all relative `require()` calls with `path.join(__dirname, '../lib/...')`; run a path audit as fork exit criterion.

---

## Implications for Roadmap

Based on the dependency chain identified across all four research files, the roadmap should follow this structure:

### Phase 1: Plugin Scaffold and Infrastructure
**Rationale:** All subsequent phases are blocked until the plugin loads correctly and the routing layer exists. The manifest placement pitfall and hook double-registration must be resolved at day one, not discovered during integration testing. `phase-state.js` must exist before any feedback loop phase is built.
**Delivers:** Plugin directory structure with `.claude-plugin/plugin.json`, `dispatch.js` skeleton, `profiles.json` with five profiles, `config.js`, empty executor stubs passing `available()`, `phase-state.js` with persistent loop counters
**Addresses:** plugin.json manifest, dispatch.js central router, per-project config, profile management commands
**Avoids:** Hook double-registration, manifest in wrong directory, in-memory loop counters
**Exit criteria:** `claude plugin validate` passes; `claude --debug` confirms manifest loaded; all six phase commands visible; loop counter survives a simulated session restart

### Phase 2: Executor Implementation (All Five Models)
**Rationale:** Executors are the root unblock for the entire pipeline. dispatch.js cannot route until all executors implement the standard interface and pass `available()` checks. Building all five together allows dispatch.js integration testing against the full model roster before phase commands are built.
**Delivers:** `codex-exec.js` (fork + adapt), `minimax-exec.js` (fork + adapt), `gemini-exec.js` (new, with 429 exponential backoff), `qwen-exec.js` (new, 180s timeout + warm-up probe + `num_ctx: 32768`), `perplexity-exec.js` (new, MCP bridge), `websearch.sh`, `fetchdocs.js`
**Avoids:** Hardcoded path references in forks, Qwen cold-start silent fallback, Gemini 429 silent failure
**Exit criteria:** Each executor passes `available()` where expected; path audit clean; cost calculation assertions against pricing table; 429-injection test passes for Gemini; warm-up probe test passes for Qwen (or documented skip if GPU not yet available)

### Phase 3: Pricing and Token Logging Infrastructure
**Rationale:** Per-provider pricing must be validated before any executor is used in a real pipeline run. Discovering negative or zero costs in Phase 5 requires corrections across all five executors. Treating this as a discrete phase forces explicit cross-provider validation.
**Delivers:** `lib/pricing.js` with nine provider-specific cost functions, `hooks/token-logger.js` with nine model schemas, `decisions.jsonl` schema with outcome signals
**Avoids:** Shared cost formula producing negative/zero costs, token logging schema divergence across nine models
**Exit criteria:** All nine models produce non-negative, non-zero costs matching the pricing table; Anthropic cache-read credits produce positive adjustment; `raw_usage` field preserved; `decisions.jsonl` receives a record on every dispatcher call

### Phase 4: Six Phase Commands and Pipeline
**Rationale:** With dispatch routing and all executors in place, the six phase commands can be wired end-to-end. Structured output schemas must be designed in this phase — they are a prerequisite for feedback loops in Phase 5. This is the first milestone where a complete pipeline run can be attempted.
**Delivers:** Six phase commands, structured phase output schemas with machine-readable markers (`<!-- STATUS: SURVIVES|FATAL_FLAW|CONDITIONAL -->`), fixed disk output paths, per-phase pipeline data flow, `session-start.js` hook
**Avoids:** Feedback loop termination evaluated from prose output (machine-readable schema designed here)
**Exit criteria:** Full end-to-end pipeline run completes on Performance profile; all six phase output files written to disk; decisions.jsonl populated with phase records including outcome signals

### Phase 5: Quality Gates and Feedback Loops
**Rationale:** Feedback loops require structured output schemas from Phase 4 and persisted loop counters from Phase 1. `checkpoint.js` requires Forge to exist. The cost-gate warning before expensive loop iterations must be implemented before any loop can run.
**Delivers:** `checkpoint.js` with runtime + static review, retry-with-feedback in Forge (max 2/task), Judge->Envision loop with hard cap, Crucible->Forge loop with hard cap, cost-gate warning before loop iteration, human escalation messages on cap exceeded
**Avoids:** Loop counter not persisted, feedback loop condition evaluated from prose, infinite loop on Crucible->Forge, biased loop termination
**Exit criteria:** Kill + restart test confirms loop counters survive; 429-cost-gate fires before loop iteration; hard cap surfaces to human with actionable message

### Phase 6: Full-Auto Run and Non-Code Support
**Rationale:** `/seraphim:run` and `--from` resume require phase-state.js to track completion across all six phases, which is stable by Phase 5. Non-code project type branching is lower complexity and natural to add here before the pipeline is considered feature-complete.
**Delivers:** `/seraphim:run` with `--from [phase]` resume, phase completion tracking, non-code project type branching in Forge and Crucible, `/seraphim:status` command
**Addresses:** Full-auto mode with resume, non-code support, status command

### Phase 7: Hook Consolidation
**Rationale:** Seven redundant v2.0 hooks must not be retired until the pipeline equivalents have been running in production long enough to be trusted. Treating consolidation as its own discrete phase ensures it does not get indefinitely deferred.
**Delivers:** Seven legacy hooks retired from `~/.claude/settings.json`, plugin hook equivalents promoted to primary, archive copies kept
**Avoids:** Old hooks still firing alongside new pipeline, double-cost entries, JSONL write conflicts
**Exit criteria:** Token log shows no entries from deprecated hook names; no double-cost entries for any pipeline session

### Phase 8: Adaptive Intelligence
**Rationale:** Pattern analysis is only meaningful after ~20 pipeline runs produce labeled decisions.jsonl data. Building this phase last ensures the data exists and the recommendation engine has signal to work with.
**Delivers:** Pattern analysis engine, auto-recommendation system with human-approval guardrail, per-phase model performance heatmap panel in dashboard, profile cost/quality comparison panel, recommendation log with accepted/rejected history
**Addresses:** Adaptive model selection, dashboard panels, recommendation guardrail

### Phase Ordering Rationale

- **Infrastructure before pipeline:** Plugin scaffold and executor errors are silent — they produce no output. Catching them early via validate and path audits costs minutes; catching them in Phase 5 costs days.
- **All executors before phase commands:** Building phases against stub executors creates a false integration picture. Real executor behavior (Qwen warm-up, Gemini rate limits, Codex JSONL parsing) only surfaces when executors are real.
- **Structured schemas before feedback loops:** Free-text phase outputs make loop condition parsing fragile. The schema design decision (machine-readable markers embedded in output files) is a cross-cutting concern that must be made once, not retrofitted after loops are built.
- **decisions.jsonl from day one:** Adaptive intelligence has zero value without data accumulated during real pipeline runs. Logging must be correct before the first pipeline run, even if the analysis layer comes eight phases later.
- **Hook consolidation late but time-boxed:** The safety net rationale is legitimate — retire hooks only after equivalents are proven. But retirement must have a concrete trigger (Phase 6 complete + N successful full pipeline runs), not "when we feel ready."

### Research Flags

Phases likely needing `/gsd:research-phase` before planning:
- **Phase 2 (Gemini executor):** `@google/generative-ai` SDK search grounding and thinking mode APIs need verification against current docs; 429 backoff correct error shape needs confirmation
- **Phase 2 (Perplexity executor):** MCP bridge pattern for calling MCP tools from a Node.js executor script is not standard; implementation mechanism needs design before building
- **Phase 5 (Cost-gate before loop iteration):** Pre-loop cost estimation approach needs implementation design
- **Phase 6 (Non-code branching):** What a "research" or "writing" project checkpoint actually verifies needs design work

Phases with standard patterns (skip research-phase):
- **Phase 1 (Plugin scaffold):** Plugin structure is documented; path is known; validate command exists
- **Phase 3 (Pricing/token logging):** Pattern is established from v2.0; extension is mechanical
- **Phase 4 (Phase commands):** Data flow is fully specified in design spec; no novel patterns
- **Phase 7 (Hook consolidation):** Settings.json editing is well-understood; the work is operational not architectural
- **Phase 8 (Dashboard panels):** Chart.js integration pattern established in v1.1; new panels follow the same generation pattern

---

## Confidence Assessment

| Area | Confidence | Notes |
|------|------------|-------|
| Stack | HIGH | Node.js v22 stack live-verified on this machine; Chart.js 4.5.1 and npm versions confirmed; `@google/generative-ai` SDK confirmed; only gap is Qwen ollama (RTX 3090 in transit) |
| Features | HIGH | Verified against PROJECT.md and design spec; all anti-features explicitly documented; competitor analysis confirms no direct analogues |
| Architecture | HIGH | Based on direct inspection of all 18 existing hooks, settings.json, and approved design spec; component responsibilities and interface contracts fully defined |
| Pitfalls | HIGH | Plugin manifest and hook migration verified against official Claude Code docs; Qwen pitfalls verified against ollama GitHub issues; feedback loop control verified against Anthropic guidance; Gemini numeric rate limits are MEDIUM (need AI Studio console check) |

**Overall confidence:** HIGH

### Gaps to Address

- **Gemini paid tier rate limits:** The research confirms 429 backoff is required but exact RPM/TPD for paid Gemini 3 Flash accounts in 2026 need verification in AI Studio console before Phase 2 begins. The free tier (10 RPM, 250 RPD) is confirmed; paid tier is higher but not pinned.
- **Perplexity MCP bridge implementation:** How to call an MCP tool from a Node.js executor script (not from a Claude session) is not fully resolved. The mechanism needs design work before Phase 2 builds `perplexity-exec.js`.
- **RTX 3090 availability:** Balanced and Budget profiles require the GPU. Phase 2 acceptance criteria should include a documented skip condition for Qwen tests until the GPU is configured. The executor must exist and fail gracefully before the GPU arrives.
- **Gemini 3.1 Pro Preview model ID stability:** Whether this model ID changes when the model goes GA is not confirmed. The model ID should be a config value, not a hardcoded string, so it can be updated without code changes.

---

## Sources

### Primary (HIGH confidence)
- `PROJECT.md` and `docs/specs/2026-04-04-seraphim-v3-design.md` — authoritative design spec
- Direct inspection of all 18 existing hooks at `~/.claude/hooks/` and `~/.claude/settings.json`
- Node.js v22.22.0 live API testing — `fs.glob()` async iterator behavior confirmed
- `npm show chart.js version` and `npm show uplot version` — version confirmation live
- Claude Code plugin documentation — plugin structure, manifest placement, validate command
- ollama GitHub issues — GPU cold-start timing, KV cache behavior, context cap enforcement
- Anthropic agent guidance — feedback loop hard caps, human escalation patterns

### Secondary (MEDIUM confidence)
- [Shipyard: Multi-agent Claude Code orchestration 2026](https://shipyard.build/blog/claude-code-multi-agent/) — multi-model orchestration patterns
- [Addy Osmani: Code Agent Orchestra](https://addyosmani.com/blog/code-agent-orchestra/) — phase specialization rationale
- [Towardsdatascience: The Multi-Agent Trap](https://towardsdatascience.com/the-multi-agent-trap/) — infinite loop failure modes
- [Agent Patterns: Infinite Agent Loop failure analysis](https://www.agentpatterns.tech/en/failures/infinite-loop) — hard cap industry standard (max 3)
- [Kinde: Orchestrating Multi-Step Agents](https://www.kinde.com/learn/ai-for-software-engineering/ai-devops/orchestrating-multi-step-agents-temporal-dagster-langgraph-patterns-for-long-running-work/) — escalation patterns
- [Google AI: Gemini 3 Developer Guide](https://ai.google.dev/gemini-api/docs/gemini-3) — search grounding, thinking mode, function calling
- [Ollama: Structured Outputs](https://docs.ollama.com/capabilities/structured-outputs) — Qwen3 JSON schema support
- [Label Your Data: LLM Orchestration strategies 2026](https://labelyourdata.com/articles/llm-fine-tuning/llm-orchestration) — profile-based routing

### Tertiary (LOW confidence — needs validation before those phases begin)
- Gemini 3 Flash paid tier rate limits — needs AI Studio console verification before Phase 2
- Perplexity Sonar MCP bridge from Node.js — needs design research before Phase 2
- Gemini 3.1 Pro Preview model ID stability — monitor as model approaches GA

---
*Research completed: 2026-04-04*
*Ready for roadmap: yes*
