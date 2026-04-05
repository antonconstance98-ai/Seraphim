# Requirements: Seraphim v3.0

**Defined:** 2026-04-04
**Core Value:** Six wings, six phases, six cognitive tasks — each assigned to the model that does it best. The human orchestrates. AI converges. Adaptive intelligence makes the system smarter over time.

## v3.0 Requirements

Requirements for the Seraphim standalone plugin. Each maps to roadmap phases.

### Plugin Infrastructure

- [x] **PLUG-01**: Plugin loads in Claude Code with `/seraphim:` namespace via plugin manifest
- [x] **PLUG-02**: `dispatch.js` routes any phase to the correct model executor based on profile, overrides, and opus_enabled flag
- [x] **PLUG-03**: `profiles.json` defines all five profiles (Performance, Balanced, Moderate, Budget, Frugal) with ten sub-phase slots each
- [x] **PLUG-04**: Per-project config at `.seraphim/config.json` stores profile, opus_enabled, overrides, and max_loops
- [x] **PLUG-05**: `config.js` reads/writes `.seraphim/config.json` with validation and defaults
- [x] **PLUG-06**: `phase-state.js` persists loop counters, retry counts, and phase completion to `.seraphim/phases/{N}/state.json` (survives session restarts)
- [x] **PLUG-07**: `models.json` defines all nine models with mechanism, pricing tier, and capability flags

### Model Executors

- [ ] **EXEC-01**: `codex-exec.js` forked from v2.0, adapted to unified interface (`execute`, `stream`, `available`), fallback chain removed (dispatch owns it)
- [ ] **EXEC-02**: `minimax-exec.js` forked from v2.0, adapted to unified interface, `runWithFallback` removed, temperature 0.01 preserved
- [ ] **EXEC-03**: `gemini-exec.js` integrates Gemini 3.1 Pro and Gemini 3 Flash via `@google/genai` SDK with search grounding, thinking mode, and 429 exponential backoff
- [ ] **EXEC-04**: `qwen-exec.js` runs Qwen locally via ollama HTTP at localhost:11434 with 180s timeout, warm-up probe, `num_ctx: 32768`, and JSON structured output for Forge mode
- [ ] **EXEC-05**: `perplexity-exec.js` integrates Perplexity Sonar via OpenAI SDK baseURL swap (`https://api.perplexity.ai`), extracts citations from `response.citations`
- [ ] **EXEC-06**: Every executor implements `execute(prompt, opts)`, `stream(prompt, opts)`, and `available()` returning consistent `{success, output, tokens, cost, error}` shape
- [ ] **EXEC-07**: `qwen-exec.js` `available()` sends a minimal inference probe (not just `/api/tags` check) and returns false gracefully when GPU/ollama is absent
- [ ] **EXEC-08**: `websearch.sh` wraps SearXNG at localhost:8888 for non-Claude models (Codex, Qwen, Gemini)
- [ ] **EXEC-09**: `fetchdocs.js` calls Context7 HTTP endpoint directly for non-Claude models

### Six Phase Pipeline

- [ ] **PIPE-01**: `/seraphim:discover` runs external (web) + internal (codebase) research tracks in parallel, writes to `.seraphim/phases/{N}/discovery/external.md` and `internal.md`
- [ ] **PIPE-02**: `/seraphim:envision` reads discovery output, generates 3-5 distinct approaches with trade-offs, writes to `.seraphim/phases/{N}/vision.md`
- [ ] **PIPE-03**: `/seraphim:judge` reads vision output, stress-tests every approach, marks each SURVIVES / FATAL_FLAW / CONDITIONAL with structured machine-readable markers, writes to `.seraphim/phases/{N}/judgment.md`
- [ ] **PIPE-04**: `/seraphim:architect` reads judgment output, creates detailed blueprint with task breakdown, dependencies, and test criteria, writes to `.seraphim/phases/{N}/blueprint.md`
- [ ] **PIPE-05**: `/seraphim:forge` executes blueprint task by task with between-task checkpoints, writes commits + `.seraphim/phases/{N}/forge-log.md`
- [ ] **PIPE-06**: `/seraphim:crucible` runs verification pass (goal-backward check) + adversarial pass (break attempt), writes to `.seraphim/phases/{N}/crucible.md`
- [ ] **PIPE-07**: Phase output schemas use structured machine-readable markers (JSON blocks or `<!-- STATUS: -->` tags) enabling feedback loop parsing
- [ ] **PIPE-08**: `/seraphim:run {N}` executes all six phases in sequence with auto-advancement and visible phase headers
- [ ] **PIPE-09**: `/seraphim:run {N} --from [phase]` resumes pipeline from a specific phase without re-running completed phases
- [ ] **PIPE-10**: Non-code project type support — blueprint declares `project_type` (code, research, writing, science, mixed); Forge and Crucible branch behavior accordingly
- [ ] **PIPE-11**: `/seraphim:new-project` initializes `.seraphim/` directory with default config for a project

### Profile Management

- [ ] **PROF-01**: `/seraphim:set-profile [name]` switches active profile and prints phase-to-model assignment table
- [ ] **PROF-02**: `/seraphim:show-profile` displays current profile with phase-to-model assignments and per-phase cost estimates
- [ ] **PROF-03**: `/seraphim:override [phase] [model]` sets a per-phase model override without changing the full profile
- [ ] **PROF-04**: `opus_enabled: false` shifts all Opus-assigned phases to profile-specific fallback models
- [ ] **PROF-05**: Balanced and Budget profiles fail gracefully with clear error when Qwen/ollama is unavailable
- [x] **PROF-06**: Users can create named custom profiles with fully custom model wiring, stored per-project in `.seraphim/profiles/`
- [x] **PROF-07**: A "naked" empty profile template is available where every model slot is unassigned and the user fills in each one
- [ ] **PROF-08**: `/seraphim:set-profile` lists both built-in (5 presets) and user-created named profiles for selection

### Quality Gates

- [ ] **QUAL-01**: Between-task checkpoint in Forge runs runtime check (tests, lint, imports) + static code review (profile's checkpoint model) on task diff
- [ ] **QUAL-02**: Checkpoint failure triggers retry-with-feedback: Forge re-runs failed task with checkpoint findings appended (max 2 retries per task)
- [ ] **QUAL-03**: Judge->Envision feedback loop: when all approaches get FATAL_FLAW, Envision re-runs with Judge's findings (max 2 loops, persisted to disk)
- [ ] **QUAL-04**: Crucible->Forge feedback loop: verification or adversarial failures trigger targeted Forge fixes with specific instructions (max 2 loops, persisted to disk)
- [ ] **QUAL-05**: When any loop cap is exceeded, pipeline stops and surfaces full findings + suggested manual resolution steps to terminal
- [ ] **QUAL-06**: `checkpoint.js` branches on `project_type`: code gets tests+lint, prose gets structure+citation check, science gets methodology+replication check

### Session & History

- [ ] **SESS-01**: `/seraphim:help` displays all commands, profiles, phase descriptions, and configuration options
- [ ] **SESS-02**: `/seraphim:history` shows pipeline run history with per-phase costs, models used, outcomes, loop counts, and total spend
- [ ] **SESS-03**: `/seraphim:pause` persists current pipeline state (completed phases, loop counts, partial outputs) to `.seraphim/phases/{N}/state.json` for multi-session work
- [ ] **SESS-04**: `/seraphim:resume {N}` restores pipeline state and continues from where it was paused
- [ ] **SESS-05**: `/seraphim:status` shows active profile, current phase progress, overrides, and model availability

### Token Logging & Cost Tracking

- [ ] **COST-01**: `lib/pricing.js` exposes per-provider cost functions for all nine models — no shared formula (prevents negative/zero costs from incompatible token schemas)
- [ ] **COST-02**: `hooks/token-logger.js` handles four incompatible token count schemas: Anthropic (`input_tokens`/`output_tokens`/`cache_read_input_tokens`), Gemini (`promptTokenCount`/`candidatesTokenCount`), MiniMax/OpenAI (`prompt_tokens`/`completion_tokens`), ollama (`prompt_eval_count`/`eval_count`)
- [ ] **COST-03**: `decisions.jsonl` logs every phase execution with: phase, model, profile, tokens_in, tokens_out, cost_usd, latency_ms, outcome, retry_count, loop_count
- [ ] **COST-04**: Quality signals captured in decisions.jsonl: crucible_pass_rate, judge_kill_rate, checkpoint_catch_rate, loop_trigger_reason
- [ ] **COST-05**: Data integrity validator checks decisions.jsonl for schema violations, missing fields, and anomalous values (negative costs, impossible token counts) on session start
- [ ] **COST-06**: `raw_usage` field preserved in token-logger records for debugging per-provider field mapping issues

### Adaptive Intelligence

- [ ] **ADPT-01**: Pattern analysis engine reads decisions.jsonl and computes per-phase model rejection rates, cost/quality ratios, and rolling performance averages
- [ ] **ADPT-02**: Auto-recommendation system suggests profile or override changes based on accumulated data (e.g., "Qwen Envision rejected by Judge 4/5 runs — consider Gemini 3.1 Pro")
- [ ] **ADPT-03**: All recommendations require explicit human approval — never auto-applied; rejected recommendations logged for audit
- [ ] **ADPT-04**: Per-phase model performance heatmap panel added to existing dashboard showing success rates by (model, phase) combination
- [ ] **ADPT-05**: Profile cost/quality comparison panel in dashboard showing average cost per run vs Crucible pass rate per profile
- [ ] **ADPT-06**: Recommendation log panel showing suggested changes, user responses (accepted/rejected/ignored), and outcome after acceptance

### Hooks Consolidation

- [ ] **HOOK-01**: Seven redundant v2.0 hooks retired atomically from `~/.claude/settings.json` after pipeline is verified: codex-review-gate, codex-plan-reviewer, codex-multi-round-reviewer, minimax-post-scan, minimax-compress, codex-router, codex-wave-validator
- [ ] **HOOK-02**: Plugin hooks (`session-start.js`, `token-logger.js`) registered in same settings.json edit that removes old hooks — never both active simultaneously
- [ ] **HOOK-03**: Archive copies of retired hooks preserved for rollback

### Multi-Project Dashboard

- [ ] **DASH-01**: Multi-project scanner discovers all `.seraphim/` directories across `~/projects/` and other configured paths
- [ ] **DASH-02**: Progress extractor parses phase-state.json, blueprint.md, and forge-log.md to surface per-project completion status (phases done, tasks remaining, current phase)
- [ ] **DASH-03**: Workflow data aggregator merges token-log.jsonl and decisions.jsonl across all projects into unified metrics
- [ ] **DASH-04**: Localhost web dashboard at `127.0.0.1:PORT` serves a self-contained HTML interface (Node.js HTTP server, no frameworks)
- [ ] **DASH-05**: Dashboard shows multi-project overview — each project's name, profile, current phase, progress bar, total cost, last activity
- [ ] **DASH-06**: Per-project drill-down shows phase roadmap, completed vs remaining tasks, model assignments, and pipeline run history
- [ ] **DASH-07**: Workflow metrics panel shows cross-project model performance, cost trends, and savings vs Opus-only baseline (extends existing Chart.js dashboard patterns)

## Future Requirements

Deferred to v4+. Tracked but not in current roadmap.

### Extended Features

- **FUTR-01**: Cost estimate before run (pre-run token budget projection per profile)
- **FUTR-02**: GUI or web dashboard for pipeline control (terminal-only in v3.0)
- **FUTR-03**: Supporting models outside the nine-model roster without manual executor authoring
- **FUTR-04**: Per-model terminal streaming across all nine models (incompatible streaming APIs)
- **FUTR-05**: ML model trained on decisions.jsonl (insufficient data at single-user scale; statistical analysis first)
- **FUTR-06**: Cross-phase feedback loops (e.g., Crucible->Envision) — creates non-linear execution

## Out of Scope

| Feature | Reason |
|---------|--------|
| Modifying Claude Code itself | Plugin API only — hooks, agents, commands |
| Running MiniMax locally | No open weights for M-2.7; API-only |
| Mobile or web interface | Terminal/CLI only per project constraints |
| Auto-applying model changes | Violates "human as orchestrator" principle |
| Real-time streaming between models | Async handoff sufficient; streaming APIs incompatible across 9 models |
| Dynamic mid-run profile switching | Makes phase outputs inconsistent |
| Loop cap above 3 | After 3 loops the problem is structural; human must intervene |
| Parallel retry branches | Multiplies cost; produces inconsistent state |
| Auto-advancing without confirmation (non-run mode) | Removes human as orchestrator; errors cascade silently |

## Traceability

Which phases cover which requirements. Updated during roadmap creation.

| Requirement | Phase | Status |
|-------------|-------|--------|
| PLUG-01 | Phase 1 | Complete |
| PLUG-02 | Phase 1 | Complete |
| PLUG-03 | Phase 1 | Complete |
| PLUG-04 | Phase 1 | Complete |
| PLUG-05 | Phase 1 | Complete |
| PLUG-06 | Phase 1 | Complete |
| PLUG-07 | Phase 1 | Complete |
| EXEC-01 | Phase 2 | Pending |
| EXEC-02 | Phase 2 | Pending |
| EXEC-03 | Phase 2 | Pending |
| EXEC-04 | Phase 2 | Pending |
| EXEC-05 | Phase 2 | Pending |
| EXEC-06 | Phase 2 | Pending |
| EXEC-07 | Phase 2 | Pending |
| EXEC-08 | Phase 2 | Pending |
| EXEC-09 | Phase 2 | Pending |
| COST-01 | Phase 2 | Pending |
| COST-02 | Phase 2 | Pending |
| COST-06 | Phase 2 | Pending |
| PIPE-01 | Phase 3 | Pending |
| PIPE-02 | Phase 3 | Pending |
| PIPE-03 | Phase 3 | Pending |
| PIPE-04 | Phase 3 | Pending |
| PIPE-05 | Phase 3 | Pending |
| PIPE-06 | Phase 3 | Pending |
| PIPE-07 | Phase 3 | Pending |
| PIPE-08 | Phase 3 | Pending |
| PIPE-09 | Phase 3 | Pending |
| PIPE-10 | Phase 3 | Pending |
| PIPE-11 | Phase 3 | Pending |
| PROF-01 | Phase 3 | Pending |
| PROF-02 | Phase 3 | Pending |
| PROF-03 | Phase 3 | Pending |
| PROF-04 | Phase 3 | Pending |
| PROF-05 | Phase 3 | Pending |
| PROF-06 | Phase 1 | Complete |
| PROF-07 | Phase 1 | Complete |
| PROF-08 | Phase 3 | Pending |
| QUAL-01 | Phase 4 | Pending |
| QUAL-02 | Phase 4 | Pending |
| QUAL-03 | Phase 4 | Pending |
| QUAL-04 | Phase 4 | Pending |
| QUAL-05 | Phase 4 | Pending |
| QUAL-06 | Phase 4 | Pending |
| COST-03 | Phase 4 | Pending |
| COST-04 | Phase 4 | Pending |
| COST-05 | Phase 4 | Pending |
| SESS-01 | Phase 5 | Pending |
| SESS-02 | Phase 5 | Pending |
| SESS-03 | Phase 5 | Pending |
| SESS-04 | Phase 5 | Pending |
| SESS-05 | Phase 5 | Pending |
| HOOK-01 | Phase 5 | Pending |
| HOOK-02 | Phase 5 | Pending |
| HOOK-03 | Phase 5 | Pending |
| ADPT-01 | Phase 6 | Pending |
| ADPT-02 | Phase 6 | Pending |
| ADPT-03 | Phase 6 | Pending |
| ADPT-04 | Phase 6 | Pending |
| ADPT-05 | Phase 6 | Pending |
| ADPT-06 | Phase 6 | Pending |
| DASH-01 | Phase 7 | Pending |
| DASH-02 | Phase 7 | Pending |
| DASH-03 | Phase 7 | Pending |
| DASH-04 | Phase 7 | Pending |
| DASH-05 | Phase 7 | Pending |
| DASH-06 | Phase 7 | Pending |
| DASH-07 | Phase 7 | Pending |

**Coverage:**
- v3.0 requirements: 68 total (58 original + 7 DASH + 3 PROF-06..08)
- Mapped to phases: 68
- Unmapped: 0

---
*Requirements defined: 2026-04-04*
*Last updated: 2026-04-04 — Phase 7 multi-project dashboard added*
