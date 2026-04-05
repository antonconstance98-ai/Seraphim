# Feature Research

**Domain:** Six-phase multi-model creative pipeline Claude Code plugin (Seraphim v3.0)
**Researched:** 2026-04-04
**Confidence:** HIGH — design spec verified against PROJECT.md; ecosystem patterns cross-verified via Claude Code plugin docs, multi-agent orchestration research (Shipyard, Addy Osmani), LLM routing literature, and 2026 practitioner sources

---

## Context: Subsequent Milestone Scope

v1.0 (Codex hook integration), v1.1 (Global Dashboard), and v2.0 (Three-Model Intelligence) are shipped. Every feature listed below is scoped to what **v3.0 Seraphim must add**. The following v2.0 components are pre-built dependencies, not features to build:

| Pre-built Component | What It Provides |
|--------------------|-----------------|
| codex-exec.js hook | Codex CLI execution; will be forked into plugin |
| minimax-exec.js hook | MiniMax API; will be forked into plugin |
| token-log.jsonl + logger | Per-call cost tracking; will be forked + extended |
| Global dashboard | Chart.js HTML dashboard; will be extended with new panels |
| Dual review gate | Stop hook pattern; being replaced by Crucible phase |
| Fallback chain logic | Codex → MiniMax → user; concept re-used in dispatch.js |

---

## Feature Landscape: Five Feature Areas

### 1. Multi-Model Orchestration Plugin

What the plugin infrastructure must provide so that six phases can run nine models.

#### Table Stakes

Features any Claude Code plugin claiming multi-model orchestration must have. Missing these = plugin doesn't function.

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| plugin.json manifest | Claude Code plugin loader requires it; without it the plugin is invisible | LOW | Standard schema: name, version, description, hooks array; one file |
| Slash command per phase | Users invoke pipeline phases via commands; no commands = no plugin experience | LOW | One `.md` file per command in `commands/`; six phase commands + set-profile + show-profile + override + status + new-project |
| Unified executor interface (`execute`, `stream`, `available`) | Every model must be callable identically; without a uniform interface, dispatch.js cannot route transparently | MEDIUM | Interface: `execute(prompt, opts) -> {success, output, tokens, cost, error}`, `stream()`, `available() -> boolean` |
| dispatch.js central router | Without centralized routing, every command hard-codes its model; profile switching is impossible | MEDIUM | Reads `.seraphim/config.json`, resolves overrides, loads executor; single abstraction layer |
| Per-project config (`.seraphim/config.json`) | Users need project-specific profiles without touching global settings | LOW | Fields: `profile`, `opus_enabled`, `overrides`; config.js handles read/write |
| Phase outputs written to fixed paths on disk | Phase outputs must persist for next phase to read; in-memory passing breaks multi-session runs | LOW | `discovery/external.md`, `discovery/internal.md`, `vision.md`, `judgment.md`, `blueprint.md`, `forge-log.md`, `crucible.md` |
| `/seraphim:status` command | Users need to verify active profile, overrides, and phase completion before running | LOW | Reads config + phase-state.js; no model calls; instant response |
| Token logging extended to all nine models | Cost visibility is a v2.0 expectation; regression if new models go untracked | MEDIUM | Fork token-logger.js into plugin; extend pricing.js with Gemini, Qwen, Perplexity rates |
| decisions.jsonl per project | Foundation for adaptive intelligence; without it the system cannot learn anything | LOW | Schema: phase, model, profile, tokens, cost, latency, outcome, retry_count, loop_count |
| Graceful failure when model is unavailable | Silently routing to an unavailable model produces corrupted phase output | LOW | `available()` check before dispatch; clear error + fallback attempt; fail-closed for execution, fail-open for review |

#### Differentiators

| Feature | Value Proposition | Complexity | Notes |
|---------|-------------------|------------|-------|
| `/seraphim:run` full-auto mode with `--from` resume | One command runs all six phases sequentially; `--from judge` resumes mid-pipeline without re-running expensive earlier phases | MEDIUM | State machine reads existing phase outputs to determine which phases are complete; skip completed phases |
| Nine distinct model executors (five profiles × breadth) | Breadth of model coverage — local + cloud + subscription — is the defining claim of Seraphim; no comparable plugin covers this range | HIGH | gemini-exec.js, qwen-exec.js, perplexity-exec.js are the three new ones; codex + minimax forked from v2.0 |
| Helper scripts for non-Claude models (`websearch.sh`, `fetchdocs.js`) | Models running outside Claude Code (Codex CLI, Qwen, Gemini) cannot use MCP tools; these bridge the gap | LOW | `websearch.sh` wraps SearXNG at localhost:8888; `fetchdocs.js` calls context7 HTTP endpoint directly |
| Hooks consolidation (remove 7 redundant v2.0 hooks) | Reduces cognitive overhead; single system to maintain; cleaner architecture | LOW | codex-review-gate, codex-plan-reviewer, codex-multi-round-reviewer, minimax-post-scan, minimax-compress, codex-router, codex-wave-validator all become redundant once pipeline is live |
| Non-code project type support | Most pipelines are code-only; supporting research/writing/science broadens addressable use case to any creative project | MEDIUM | Project type declared in Architect blueprint; Forge and Crucible behavior branches on `project_type` field |

#### Anti-Features

| Feature | Why Requested | Why Problematic | Alternative |
|---------|---------------|-----------------|-------------|
| Auto-advancing phases without visible confirmation | Faster end-to-end runs | Removes human as orchestrator; errors cascade silently across phases | Explicit phase commands per phase + `/seraphim:run` for full-auto with visible phase headers and completion summaries |
| Per-model terminal streaming | Feels responsive | Different streaming APIs across nine models; Qwen local has no streaming; adds four integration surface areas | Spinner during model call + structured completion summary; Claude subagents stream natively as a side-effect |
| GUI or web dashboard for pipeline control | Richer UX | Out of scope (terminal/CLI only per PROJECT.md); adds server dependency; one more thing to maintain | Extend existing Chart.js dashboard HTML for metrics; terminal output for pipeline status |
| Supporting models outside the nine-model roster | Flexibility | Explicit out-of-scope constraint in PROJECT.md; adds executor surface area mid-milestone | Custom profile in config.json can point to any executor; user adds executor file manually |

---

### 2. Adaptive Model Selection and Learning

What the system must do to learn from real usage and improve model routing over time.

#### Table Stakes

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| decisions.jsonl schema with outcome signals | Without outcome labels (did Crucible pass? how many loops?), there is nothing to learn from — logging without signals is inert | LOW | Required fields: phase, model, profile, tokens_in, tokens_out, cost_usd, latency_ms, outcome (success/failure/retry), retry_count, loop_count, quality_signals (crucible_pass, judge_kill_rate) |
| Human-approval guardrail on all recommendations | Auto-applying model changes without user consent is explicitly out of scope per PROJECT.md; violating this makes the system unsafe | LOW | Recommendations printed to terminal at session start; user must run `/seraphim:override` or manually edit config to apply |
| Pattern analysis command or periodic script | Raw JSONL is not actionable; the analysis layer turns logs into recommendations like "Qwen Envision rejected by Judge 4/5 times" | MEDIUM | Runs on session start (lightweight) or via explicit command; reads decisions.jsonl; statistical analysis only — no ML |

#### Differentiators

| Feature | Value Proposition | Complexity | Notes |
|---------|-------------------|------------|-------|
| Phase-level rejection rate tracking | Pinpoints exactly which model in which phase is underperforming; more precise than any existing routing tool | LOW | Computed from decisions.jsonl: loop_count > 0 grouped by (phase, model); surfaces patterns no benchmark can reveal |
| Per-phase model performance heatmap in dashboard | Visualization of which models succeed per phase from real usage data; unique to Seraphim | MEDIUM | Extend dashboard.html with new panel; data sourced from aggregated decisions.jsonl |
| Profile cost/quality ratio tracking | Tells users if their chosen profile actually delivers better quality per dollar over time; enables informed profile switching | MEDIUM | Aggregate by profile: avg cost per run vs Crucible pass rate per run; render as profile comparison panel in dashboard |
| Recommendation log with accepted/rejected history | Shows users what the system suggested and what they chose; builds trust via transparency; tracks whether recommendations improved outcomes | LOW | Append `type: "recommendation"` records to decisions.jsonl; capture user response (accepted/rejected/ignored) |

#### Anti-Features

| Feature | Why Requested | Why Problematic | Alternative |
|---------|---------------|-----------------|-------------|
| Auto-apply model changes when confidence is high | Faster improvement | Violates "human as orchestrator" principle; confidence scoring can be wrong; silent quality regressions undetectable | Recommendations only; require explicit user action to apply; log all rejected recommendations for audit |
| Real-time routing adjustment during a pipeline run | Maximum adaptivity | Model assignment changes between Envision and Judge makes outputs inconsistent; debugging becomes impossible | Apply recommendations only at start of next pipeline run; never mid-run |
| ML model trained on decisions.jsonl | Richer signal | 100-500 samples at single-user scale is insufficient for reliable ML; adds sklearn/torch dependency and training pipeline for negligible benefit over statistical analysis | Statistical analysis (rates, ratios, rolling averages) on JSONL; graduate to ML only if record count exceeds 1000 labeled outcomes |

---

### 3. Profile-Based Cost Management

What the five-profile system must provide so users can control cost vs quality.

#### Table Stakes

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| Five profiles with pre-assigned models per phase | Without profiles, every user must configure nine phase-to-model assignments manually; friction kills adoption | LOW | profiles.json: Performance, Balanced, Moderate, Budget, Frugal; each defines all ten sub-phase slots |
| `/seraphim:set-profile [name]` command | Profiles are useless without a one-command switch | LOW | Writes to `.seraphim/config.json`; prints new phase-to-model assignment table |
| `/seraphim:show-profile` command | Users need to verify configuration before committing to expensive runs | LOW | Reads config + profiles.json; renders phase-to-model table with cost estimates per phase |
| Per-phase override (`overrides` in config.json) | Power users need to deviate from a profile for one phase without creating a full custom config | LOW | `overrides: { "architect": "sonnet-4.6" }` in config; dispatch.js checks overrides before profile lookup |
| opus_enabled toggle with per-profile fallbacks | Opus is the most expensive model ($25/Mtok out); billing-sensitive users need a safe off-ramp | LOW | Flag in config.json; per-profile fallback chain in profiles.json; dispatch checks flag before assigning Opus phases |
| Graceful unavailability for GPU-dependent profiles | Balanced and Budget profiles require local Qwen; running them without GPU must fail clearly, not silently | LOW | `available()` in qwen-exec.js checks ollama at localhost:11434; dispatch returns clear error: "Balanced profile unavailable: Qwen not running" |

#### Differentiators

| Feature | Value Proposition | Complexity | Notes |
|---------|-------------------|------------|-------|
| `profile: "custom"` with explicit per-sub-phase assignment | Power users define exactly which model runs each of ten sub-phase slots; no comparable plugin offers this granularity | LOW | JSON schema already defined in design spec; dispatch reads `phases` object when profile is "custom" |
| Cost estimate before run | Users see approximate cost before committing; prevents surprise bills on long pipeline runs | MEDIUM | Pre-run: sum per-phase model costs based on typical token budgets (configurable per project type); rendered before first model call |
| Balanced and Budget profiles at near-zero cost via local Qwen | Zero-cost execution on owned hardware; 50-80% cost reduction vs cloud-only profiles | HIGH | Requires ollama + Qwen 3.5-27B Q4 + RTX 3090; GPU in transit; gate with `available()` check |

#### Anti-Features

| Feature | Why Requested | Why Problematic | Alternative |
|---------|---------------|-----------------|-------------|
| Dynamic mid-run profile switching | Respond to cost overruns in real time | Makes phases inconsistent: Judge output generated by one model, Architect reasoning against a different one | Hard-set profile at run start; show cost estimate before run so user selects correctly upfront |
| Automatic profile downgrade on budget threshold | Protect against surprise bills | Silently degrading quality without user knowledge violates the orchestrator principle; user may not notice | Alert user when approaching daily budget; pause pipeline and ask which profile to use for remaining phases |
| More than five profiles | More granularity | Five profiles cover the full cost/quality spectrum (Performance → Frugal); more profiles increase cognitive load without increasing coverage | Custom profile covers any configuration not served by the five presets |

---

### 4. Between-Task Checkpoint System (Forge Phase)

What the Forge phase checkpoint must provide to catch errors before they compound across tasks.

#### Table Stakes

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| Runtime check before advancing to next task | Without a sanity check, broken code accumulates silently; bugs compound across tasks and produce unfixable final state | MEDIUM | checkpoint.js: run test suite, lint, check imports; zero-exit means pass; non-zero means fail |
| Static code review on task diff | Runtime checks miss logic errors, security issues, off-by-ones; static review catches what tests don't cover | MEDIUM | Profile's checkpoint static model (e.g. MiniMax in Performance) reviews git diff of current task; returns structured pass/fail + findings |
| Retry-with-feedback on checkpoint failure | Proceeding past a failed checkpoint is worse than no checkpoint; feedback must feed back | MEDIUM | Forge executor re-runs the failed task with checkpoint findings appended to prompt; max 2 retries per task |
| Checkpoint results logged to forge-log.md | Audit trail for Crucible verification; without log, Crucible cannot know what was checked and what was found | LOW | Log per task: model, outcome, findings, retry count; append-only |
| Hard cap at 2 retries per task before human escalation | Infinite retry loops are the #1 production failure mode for LLM agents (confirmed by Towardsdatascience, Agent Patterns, Medium/LLM tool-calling 2026) | LOW | Retry counter in phase-state.js; on cap exceeded: surface task + findings to terminal with "manual resolution required" message |

#### Differentiators

| Feature | Value Proposition | Complexity | Notes |
|---------|-------------------|------------|-------|
| Two-reviewer checkpoint (runtime model + static model, separate) | Runtime (test runner + Sonnet) and static (MiniMax adversarial) catch different failure modes; combined catch rate exceeds either alone | MEDIUM | Both must pass to advance; either failing triggers retry; design spec defines this split at the sub-phase level |
| Checkpoint model assignment configurable per profile | Budget profile uses Qwen for both; Performance uses Sonnet + MiniMax; not locked to expensive models | LOW | Resolved via dispatch.js using `forge_checkpoint_runtime` and `forge_checkpoint_static` as first-class phase keys in profiles.json |
| Non-code checkpoint branching | Research/writing projects need different verification (citation coherence, argument structure) than code projects | MEDIUM | checkpoint.js branches on `project_type` from blueprint.md; code: lint+tests; prose: structure check + citation validation |

#### Anti-Features

| Feature | Why Requested | Why Problematic | Alternative |
|---------|---------------|-----------------|-------------|
| Checkpoint after every file write | Maximum granularity | Prohibitively expensive at $0.80-3/Mtok for checkpoint models; creates latency that makes Forge unusable for large tasks | Checkpoint between logical tasks as defined in blueprint; not between individual file edits |
| Automatic git rollback on checkpoint failure | Prevents broken state | Destroys partial work that may be partially correct and useful for retry context; user loses diff for analysis | Retry with feedback; human escalation on cap exceeded; user decides whether to roll back |
| Streaming checkpoint output to terminal | Feels responsive | MiniMax and Qwen do not support streaming in the same way as Claude; inconsistent UX across profiles | Spinner during check + structured result summary on completion |

---

### 5. Feedback Loops with Hard Caps

What the Judge->Envision and Crucible->Forge loops must provide to enable self-correction without infinite recursion.

#### Table Stakes

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| Judge->Envision loop when all approaches fail | Without retry, a "none survive" verdict dead-ends the pipeline with no resolution path | MEDIUM | Judge output contains structured signal: `{"status": "all_failed", "findings": [...]}` ; orchestrator re-runs Envision with findings appended |
| Crucible->Forge loop on verification failure | Without retry, Crucible is a reporting tool; verification failures must trigger targeted fixes | MEDIUM | Crucible output includes specific fix instructions per failed requirement; Forge re-runs identified tasks only, not the full phase |
| Crucible->Forge loop on adversarial findings | Adversarial pass has no effect if findings cannot trigger fixes | MEDIUM | Adversarial findings are distinct from verification failure; Forge receives targeted patch instructions per finding |
| Hard cap of 2 loops on all feedback cycles | Infinite loops are the critical production failure mode; multiple 2026 sources confirm "max 3 retries then escalate to human" as the industry standard pattern | LOW | Loop counter in phase-state.js; on cap exceeded: stop, surface full findings to terminal, message "manual resolution required — pipeline cannot self-resolve" |
| Human escalation message with actionable content on cap exceeded | Users need to know what the unresolved problem is, not just that it failed | LOW | Terminal output: which loop hit cap, full findings from last phase run, suggested manual steps (e.g., "restart from Architect with a revised approach") |
| Feedback context appended to retry prompt | Retrying without feeding the failure reason produces the same output again | LOW | Each loop appends prior phase's structured findings to the next run's prompt; format: `## Previous [Phase] Findings\n[structured findings]` |

#### Differentiators

| Feature | Value Proposition | Complexity | Notes |
|---------|-------------------|------------|-------|
| Structured feedback format between phases | Free-text feedback produces inconsistent retry results; structured schema (FATAL_FLAW, CONDITION, MISSING_REQUIREMENT) enables targeted retries instead of re-running everything | MEDIUM | Judge and Crucible outputs use fixed JSON schema; Envision and Forge parsers read fields explicitly; design spec defines schemas |
| Loop count logged to decisions.jsonl | Enables adaptive intelligence to surface patterns: "Envision needed 2 loops in 6/8 runs with Qwen" — a recommendation trigger | LOW | Add `loop_count` field to decisions.jsonl schema; logged per phase at completion |
| Configurable loop cap per project (1-3 range) | Some projects tolerate more iteration; tight deadlines need cap at 1 | LOW | `max_loops` in `.seraphim/config.json`; defaults to 2; validate range 1-3; budget profiles benefit from cap at 1 |

#### Anti-Features

| Feature | Why Requested | Why Problematic | Alternative |
|---------|---------------|-----------------|-------------|
| Loop cap above 3 | More chances to self-resolve | After 3 loops with the same models, the problem is structural; the model cannot fix what it cannot see; confirmed by agent loop research | Hard cap at 3; surface to human; let human decide if re-architecting is needed before re-running |
| Cross-phase feedback loops (e.g., Crucible->Envision) | Richer self-correction — go back further if root cause is in approach design | Creates non-linear execution; phase outputs become inconsistent; debugging becomes intractable | Keep loops local: Judge->Envision and Crucible->Forge only; cross-phase issues require human to run `/seraphim:run --from [phase]` |
| Parallel retry branches (run Envision N times simultaneously, pick best) | Higher quality through diversity | Multiplies cost by N; produces multiple inconsistent vision.md files in state; Judge cannot receive N versions cleanly | Sequential retry with feedback; Judge picks the best surviving approach in a single run, not the system |

---

## Feature Dependencies

```
plugin.json manifest
    └──required by──> All slash commands
    └──required by──> All hooks

dispatch.js + profiles.json
    └──required by──> All six phase commands
    └──required by──> Feedback loops (loop re-invokes phase commands)
    └──required by──> Checkpoint system (checkpoint models resolved via dispatch)

.seraphim/config.json + config.js
    └──required by──> dispatch.js
    └──required by──> Profile management commands
    └──required by──> max_loops config

Nine model executors
    └──required by──> dispatch.js
    (codex-exec + minimax-exec: fork from v2.0)
    (gemini-exec + qwen-exec + perplexity-exec: new builds)

phase-state.js (phase progress tracker)
    └──required by──> /seraphim:run full-auto
    └──required by──> --from flag resume
    └──required by──> Hard cap enforcement (loop counters + retry counters)

checkpoint.js (runtime + static review)
    └──required by──> Forge phase
    └──required by──> Crucible->Forge feedback loop (checkpoint determines if fix worked)

Structured phase output schemas (JSON fields in vision.md, judgment.md, crucible.md)
    └──required by──> Judge->Envision feedback loop
    └──required by──> Crucible->Forge feedback loop
    └──required by──> Pattern analysis engine (needs machine-parseable outcomes)

decisions.jsonl schema + per-phase logging
    └──required by──> Pattern analysis / adaptive intelligence
    └──required by──> Per-phase performance heatmap (dashboard)
    └──required by──> Profile cost/quality comparison (dashboard)
    └──required by──> Loop count tracking

RTX 3090 + ollama + Qwen 3.5-27B Q4
    └──required by──> Balanced profile
    └──required by──> Budget profile
    (GPU in transit; these two profiles unavailable until qwen-exec.js available() returns true)

Forge phase (complete, with checkpoints)
    └──required by──> Crucible phase
    └──required by──> Crucible->Forge feedback loop

Judge phase (complete, with structured output schema)
    └──required by──> Judge->Envision feedback loop
```

### Dependency Notes

- **Nine executors are the root unblock for the entire pipeline.** Without all nine executors passing `available()` checks, dispatch.js cannot route any phase. Executors must be the first deliverable.
- **Structured phase output schemas must be designed before feedback loops.** Free-text phase outputs make loop context injection unreliable. Schema design (what fields does judgment.md contain? what does crucible.md return?) must precede loop implementation.
- **decisions.jsonl must be populated from day one.** The adaptive intelligence features have zero value without accumulated data. The logging schema must be correct before the first pipeline run, even if the analysis layer comes later in the milestone.
- **Balanced and Budget profiles must fail gracefully without GPU.** qwen-exec.js `available()` must check ollama at `localhost:11434` and return false when GPU/ollama is absent. dispatch.js must surface this as a clear error, not silently route to a broken executor.
- **Checkpoint system depends on profile config being finalized.** Which model runs runtime vs static checkpoints is resolved by dispatch.js reading the profile. Profile definitions must be stable before checkpoint model assignment is tested end-to-end.

---

## MVP Definition

### Launch With (v3.0 Phase 1-6: Plugin Core + Pipeline)

Minimum to run a complete six-phase pipeline end-to-end on any profile.

- [ ] plugin.json manifest and directory structure — plugin must load
- [ ] dispatch.js + profiles.json (five profiles; overrides supported; opus_enabled toggle)
- [ ] All five executor forks/builds: codex-exec.js (fork), minimax-exec.js (fork), gemini-exec.js (new), qwen-exec.js (new, with GPU unavailability guard), perplexity-exec.js (new)
- [ ] Six phase commands with fixed output paths and disk persistence
- [ ] Per-project config (.seraphim/config.json) with profile + opus_enabled + overrides
- [ ] Token logging extended to all nine models (fork + extend pricing.js)
- [ ] decisions.jsonl schema defined and logging at each phase completion (outcome signals required from day one)
- [ ] helper scripts: websearch.sh, fetchdocs.js

### Add After Core Pipeline Works (v3.0 Phase 7-10: Quality Gates + Loops)

- [ ] Between-task checkpoints in Forge: checkpoint.js with runtime check + static review
- [ ] Retry-with-feedback in Forge (max 2 retries per task, hard cap)
- [ ] Judge->Envision feedback loop with structured feedback format + hard cap
- [ ] Crucible->Forge feedback loop (both verification and adversarial) with hard cap
- [ ] `/seraphim:run` full-auto mode with `--from` resume flag
- [ ] Non-code project type branching in Forge and Crucible
- [ ] phase-state.js with loop counters and human escalation messages

### Add After Pipeline Is Stable (v3.0 Phase 11+: Intelligence + Polish)

- [ ] Pattern analysis engine (reads decisions.jsonl; produces recommendations; requires ~20+ runs of data)
- [ ] Auto-recommendation system with human-approval guardrail
- [ ] Per-phase model performance heatmap in dashboard
- [ ] Profile cost/quality comparison panel in dashboard
- [ ] Recommendation log with accepted/rejected history
- [ ] Hooks consolidation (remove 7 redundant v2.0 hooks after pipeline is verified as replacement)
- [ ] Cost estimate before run (pre-run token budget projection)

---

## Feature Prioritization Matrix

| Feature | User Value | Implementation Cost | Priority |
|---------|------------|---------------------|----------|
| plugin.json + slash commands | HIGH | LOW | P1 |
| dispatch.js + profiles.json | HIGH | MEDIUM | P1 |
| All nine model executors | HIGH | HIGH | P1 |
| Per-project config + profile commands | HIGH | LOW | P1 |
| Phase outputs written to disk | HIGH | LOW | P1 |
| Token logging extended to 9 models | HIGH | MEDIUM | P1 |
| decisions.jsonl logging with outcome signals | HIGH | LOW | P1 |
| Helper scripts (websearch.sh, fetchdocs.js) | MEDIUM | LOW | P1 |
| Between-task checkpoints (Forge) | HIGH | MEDIUM | P1 |
| Retry-with-feedback in Forge | HIGH | MEDIUM | P1 |
| Feedback loops + hard caps (Judge, Crucible) | HIGH | MEDIUM | P1 |
| phase-state.js + human escalation on cap exceeded | HIGH | LOW | P1 |
| `/seraphim:run` full-auto + `--from` resume | MEDIUM | MEDIUM | P2 |
| Per-phase overrides + custom profile | MEDIUM | LOW | P2 |
| Non-code project type branching | MEDIUM | MEDIUM | P2 |
| Structured phase output schemas | HIGH | LOW | P2 (design precedes loops) |
| Pattern learning engine | MEDIUM | MEDIUM | P2 |
| Auto-recommendations + guardrail | MEDIUM | LOW | P2 |
| Dashboard heatmap + profile comparison | MEDIUM | MEDIUM | P2 |
| Hooks consolidation | LOW | LOW | P2 |
| Cost estimate before run | LOW | MEDIUM | P3 |
| Configurable loop cap | LOW | LOW | P3 |

---

## Competitor Feature Analysis

No direct competitors exist for a six-phase multi-model Claude Code plugin. Closest analogues:

| Feature | Agent Teams (Anthropic native) | ruflo / oh-my-claudecode | Seraphim v3.0 |
|---------|-------------------------------|--------------------------|---------------|
| Phase structure | None (flat task list) | 4-5 step pipelines typically | Six phases with cognitive specialization (diverge → evaluate → converge → execute → verify) |
| Multi-model routing | Claude-only (subagents) | Claude + Codex typically | Nine models across five profiles; local + cloud + subscription |
| Cost profiles | None | None | Five presets + per-phase override + custom profile |
| Adaptive intelligence | None | None | decisions.jsonl logging + pattern analysis + human-approved recommendations |
| Feedback loops | Implicit (retry in prompt) | Ad-hoc | Explicit structured loops with hard caps and human escalation |
| Checkpoint system | None | None | Two-reviewer between-task checkpoints (runtime + static) |
| Non-code support | Yes (general) | Typically code-focused | Explicit project_type branching in Forge and Crucible |
| Local model support | No | Rarely | Qwen 3.5-27B via ollama (GPU required; graceful degradation without) |
| Plugin architecture | Native (Agent Teams API) | Plugin/hook based | Standalone plugin; no GSD or Superpowers runtime dependency |

---

## Sources

- [Shipyard: Multi-agent Claude Code orchestration in 2026](https://shipyard.build/blog/claude-code-multi-agent/)
- [Addy Osmani: The Code Agent Orchestra — what makes multi-model coding work](https://addyosmani.com/blog/code-agent-orchestra/)
- [Label Your Data: LLM Orchestration strategies and best practices 2026](https://labelyourdata.com/articles/llm-fine-tuning/llm-orchestration)
- [Towardsdatascience: The Multi-Agent Trap (infinite loop failure modes)](https://towardsdatascience.com/the-multi-agent-trap/)
- [Agent Patterns: Infinite Agent Loop failure analysis](https://www.agentpatterns.tech/en/failures/infinite-loop)
- [Medium (Komalbaparmar): LLM Tool-Calling in Production — the Infinite Loop failure mode](https://medium.com/@komalbaparmar007/llm-tool-calling-in-production-rate-limits-retries-and-the-infinite-loop-failure-mode-you-must-2a1e2a1e84c8)
- [Kinde: Orchestrating Multi-Step Agents — hard caps and escalation patterns 2026](https://www.kinde.com/learn/ai-for-software-engineering/ai-devops/orchestrating-multi-step-agents-temporal-dagster-langgraph-patterns-for-long-running-work/)
- [Atlosz: LLM Cost Optimization and Multi-Model Routing — profile-based approaches](https://atlosz.hu/en/blog/llm-koltsegoptimalizalas-routing-strategia/)
- [Mavik Labs: LLM Cost Optimization 2026 — routing, caching, batching](https://www.maviklabs.com/blog/llm-cost-optimization-2026)
- [Ollama: Structured Outputs documentation — Qwen3 JSON schema support](https://docs.ollama.com/capabilities/structured-outputs)
- [Google AI: Gemini 3 Developer Guide — search grounding + function calling + thinking mode](https://ai.google.dev/gemini-api/docs/gemini-3)
- [Perplexity: Sonar MCP integration — citations pipeline](https://mcp.directory/servers/perplexity)
- [Claude Code: Create plugins — official plugin structure documentation](https://code.claude.com/docs/en/plugins)
- [arxiv 2512.18388: Divergent/convergent thinking scaffolding in human-AI co-creation](https://arxiv.org/html/2512.18388v1)
- [IxDF: Convergent Thinking — design pattern reference](https://ixdf.org/literature/topics/convergent-thinking)
- Project context: `/home/alucard/projects/Seraphim/.planning/PROJECT.md`
- Design spec: `/home/alucard/projects/Seraphim/docs/specs/2026-04-04-seraphim-v3-design.md`

---

*Feature research for: Six-phase multi-model creative pipeline Claude Code plugin (Seraphim v3.0)*
*Researched: 2026-04-04*
