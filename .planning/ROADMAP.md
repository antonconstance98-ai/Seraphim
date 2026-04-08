# Roadmap: Seraphim

## Milestones

- ✅ **v1.0 Claude X Codex** — Phases 1-4 (shipped 2026-04-02) — [archive](milestones/v1.0-ROADMAP.md)
- ✅ **v1.1 Global Metrics Dashboard** — Phases 5-7 (shipped 2026-04-03) — [archive](milestones/v1.1-ROADMAP.md)
- ✅ **v2.0 Three-Model Intelligence** — Phases 8-14 (shipped 2026-04-03)
- ✅ **v2.0 Adaptive Intelligence** — Phase 15 (shipped 2026-04-04)
- 🔄 **v3.0 Seraphim** — Phases 1-11 (in progress) — clean break, phase numbering reset

---

## v3.0 Seraphim

**Milestone goal:** Replace the hook-based multi-model system with a standalone Claude Code plugin delivering a six-phase creative pipeline across nine models and five cost profiles. Adaptive intelligence learns from accumulated run data.

## Phases

- [ ] **Phase 1: Plugin Scaffold and Infrastructure** — Plugin loads, dispatch routes, config persists, phase state survives restarts
- [ ] **Phase 2: Model Executors and Pricing** — All nine models callable through uniform interface; token costs validated per provider
- [ ] **Phase 3: Six Phase Pipeline and Profile Management** — End-to-end pipeline runs; all profile and override commands work
- [ ] **Phase 4: Quality Gates and Decision Logging** — Checkpoints catch failures; feedback loops run with hard caps; decisions are logged
- [ ] **Phase 5: Session Commands and Hook Consolidation** — Full-auto run with resume; all session commands work; seven legacy hooks retired
- [ ] **Phase 6: Adaptive Intelligence** — Pattern analysis produces recommendations; dashboard panels show per-phase performance
- [ ] **Phase 7: Multi-Project Dashboard** — Localhost web interface aggregates progress, metrics, and workflow data across all Seraphim-managed projects
- [ ] **Phase 8: Thought Orphanage Integration** — Slash command for capturing seed thoughts at project and global level, braindump workflow, dashboard representation
- [ ] **Phase 9: Human-AI Cognitive Division** — Research where AI vs human leverage is best placed in the Seraphim pipeline, informed by divergent/convergent cognition research
- [ ] **Phase 10: Context Management and Token Optimization** — Research and implement strategies to reduce token usage across the nine-model pipeline
- [ ] **Phase 11: OpenClaw Local RAG Integration** — Research OpenClaw's architecture and adapt its local RAG system for project knowledge referencing in Seraphim

## Phase Details

### Phase 1: Plugin Scaffold and Infrastructure
**Goal**: The plugin loads in Claude Code, dispatch routes to the correct executor, per-project config persists, and phase state survives session restarts
**Depends on**: Nothing (first phase)
**Requirements**: PLUG-01, PLUG-02, PLUG-03, PLUG-04, PLUG-05, PLUG-06, PLUG-07
**Success Criteria** (what must be TRUE):
  1. Running `claude plugin validate` on the plugin directory passes with no errors, and all `/seraphim:` commands appear in Claude Code's command list
  2. `dispatch.js` resolves a model assignment correctly at all three levels — override wins over opus_enabled toggle which wins over profile preset — verifiable by setting each level and observing the resolved model
  3. Creating a new project with `/seraphim:new-project` produces a `.seraphim/config.json` file with correct defaults, readable and writable by `config.js` without error
  4. After simulating a session restart (killing the process mid-phase), `phase-state.js` restores loop counters and completion flags from `.seraphim/phases/{N}/state.json` with no data loss
  5. `models.json` contains all nine models with mechanism, pricing tier, and capability flags — verifiable by reading the file and confirming all nine entries
**Plans:** 3 plans
Plans:
- [x] 01-01-PLAN.md — Plugin directory scaffold, manifest, models.json, profiles.json
- [x] 01-02-PLAN.md — Core modules: config.js, dispatch.js, phase-state.js
- [x] 01-03-PLAN.md — new-project command, session-start hook, plugin validation

### Phase 2: Model Executors and Pricing
**Goal**: All nine models are callable through a uniform interface; token costs are validated against the pricing table per provider with no shared formula
**Depends on**: Phase 1
**Requirements**: EXEC-01, EXEC-02, EXEC-03, EXEC-04, EXEC-05, EXEC-06, EXEC-07, EXEC-08, EXEC-09, COST-01, COST-02, COST-06
**Success Criteria** (what must be TRUE):
  1. Each executor returns `{success, output, tokens, cost, error}` from `execute()` — calling any executor directly and inspecting the return shape confirms the contract
  2. `available()` for the Qwen executor sends a real inference probe (not just an `/api/tags` check) and returns `false` gracefully when ollama is absent, without throwing or hanging
  3. All nine models produce non-negative, non-zero cost calculations matching the pricing table; Anthropic cache-read produces a positive credit adjustment; `raw_usage` is preserved in every record — verifiable by running a test call through each executor
  4. Running the path audit on forked codex-exec.js and minimax-exec.js shows no old absolute path references — all `require()` calls resolve relative to `__dirname`
  5. `websearch.sh` queries SearXNG at localhost:8888 and returns results; `fetchdocs.js` calls the Context7 endpoint and returns documentation
**Plans:** 5 plans
Plans:
- [x] 02-01-PLAN.md — Per-provider pricing module and token logger with four-schema normalization
- [x] 02-02-PLAN.md — Fork codex-exec.js and minimax-exec.js to unified interface
- [x] 02-03-PLAN.md — Gemini executor with search grounding and thinking mode
- [x] 02-04-PLAN.md — Qwen local executor and Perplexity dual-path executor
- [x] 02-05-PLAN.md — Helper scripts: websearch.sh and fetchdocs.js

### Phase 3: Six Phase Pipeline and Profile Management
**Goal**: A full six-phase pipeline run completes end-to-end on Performance profile; all profile switching and override commands work correctly
**Depends on**: Phase 2
**Requirements**: PIPE-01, PIPE-02, PIPE-03, PIPE-04, PIPE-05, PIPE-06, PIPE-07, PIPE-08, PIPE-09, PIPE-10, PIPE-11, PROF-01, PROF-02, PROF-03, PROF-04, PROF-05
**Success Criteria** (what must be TRUE):
  1. Running `/seraphim:run {N}` on Performance profile executes all six phases in sequence and writes all six output files (`external.md`, `internal.md`, `vision.md`, `judgment.md`, `blueprint.md`, `forge-log.md`, `crucible.md`) to `.seraphim/phases/{N}/`
  2. `judgment.md` contains machine-readable markers (`SURVIVES`, `FATAL_FLAW`, or `CONDITIONAL`) that a script can parse without reading prose — verifiable by grepping the output file
  3. Running `/seraphim:set-profile balanced` prints the phase-to-model assignment table, and running `/seraphim:override judge gemini-flash` changes only the Judge assignment without altering other phases
  4. Setting `opus_enabled: false` in config causes all Opus-assigned phases to use profile-specific fallback models — verifiable by checking which executor is called for Envision and Architect
  5. Running `/seraphim:run {N} --from judge` skips Discover and Envision and resumes from Judge without re-running completed phases
  6. Running on a non-code project (blueprint declares `project_type: research`) causes Forge and Crucible to use prose-appropriate behavior instead of test+lint
**Plans:** 6 plans
Plans:
- [x] 03-01-PLAN.md — Shared infrastructure: dispatch CLI entry point, marker parser, wing banners
- [x] 03-02-PLAN.md — Discover and Envision commands (Wings I-II)
- [x] 03-03-PLAN.md — Judge and Architect commands (Wings III-IV)
- [x] 03-04-PLAN.md — Forge and Crucible commands (Wings V-VI)
- [x] 03-05-PLAN.md — Profile management: set-profile, show-profile, override
- [x] 03-06-PLAN.md — Run orchestrator with --from resume support
**UI hint**: yes

### Phase 03.1: Parallel Discovery Research Tracks (INSERTED)

**Goal:** Enhance the Discover phase (Wing I) to spawn multiple parallel research agents hitting Perplexity from different angles — existing solutions, anti-patterns, architecture patterns, dependencies — then merge results into a single external.md for richer discovery output
**Requirements**: PIPE-01
**Depends on:** Phase 3
**Success Criteria** (what must be TRUE):
  1. `/seraphim:discover` spawns 3-4 parallel research agents, each focused on a different research dimension (solutions/frameworks, anti-patterns/pitfalls, architecture/patterns, dependencies/compatibility)
  2. Each parallel agent uses Perplexity Sonar (or the profile's discover_external model) for web-grounded research
  3. Results from all parallel agents are merged into a single structured `external.md` with per-track sections and source attribution
  4. If any parallel track fails, the remaining tracks still complete and external.md notes the failed track with a TRACK_FAILED stub
  5. The internal codebase analysis track remains unchanged and runs independently of the parallel external tracks
**Plans:** 1 plan

Plans:
- [ ] 03.1-01-PLAN.md — Rewrite discover.md Step 6 for 4 parallel external research tracks; update seraphim-discover agent

### Phase 4: Quality Gates and Decision Logging
**Goal**: Forge checkpoints catch task failures and trigger retry-with-feedback; feedback loops run with persisted hard caps; every phase execution is logged to decisions.jsonl with outcome signals
**Depends on**: Phase 3
**Requirements**: QUAL-01, QUAL-02, QUAL-03, QUAL-04, QUAL-05, QUAL-06, COST-03, COST-04, COST-05
**Success Criteria** (what must be TRUE):
  1. After a Forge task fails its checkpoint, Forge re-runs the task with the checkpoint findings appended — the retry is visible in `forge-log.md` with the failure reason — and the task does not retry more than twice
  2. When all Envision approaches receive `FATAL_FLAW`, the Judge->Envision loop triggers, Envision re-runs with Judge's findings, and the loop counter in `.seraphim/phases/{N}/state.json` increments — verifiable by killing the session mid-loop and restarting: the counter is preserved
  3. When any loop cap is exceeded, the pipeline stops and prints the full findings with suggested manual resolution steps to terminal — it does not silently proceed or retry
  4. `checkpoint.js` branches correctly on `project_type`: a code project triggers tests+lint; a prose project triggers structure+citation check; a science project triggers methodology+replication check — verifiable by running each type
  5. After a complete pipeline run, `decisions.jsonl` contains one record per phase with phase, model, profile, tokens_in, tokens_out, cost_usd, latency_ms, outcome, retry_count, loop_count, and at least one quality signal field — verifiable by reading the file and checking schema
  6. The data integrity validator detects a manually injected negative-cost record in decisions.jsonl on the next session start and reports the violation
**Plans:** 4 plans

Plans:
- [x] 04-01-PLAN.md — checkpoint.js, decisions-logger.js, decisions-validator.js (new infrastructure)
- [x] 04-02-PLAN.md — forge.md: checkpoint gate, retry-with-feedback, crucible fix mode
- [x] 04-03-PLAN.md — envision.md: loop path; judge.md: decisions logging + escalation
- [x] 04-04-PLAN.md — crucible.md: fix instructions + logging; session-start: validator

### Phase 5: Session Commands and Hook Consolidation
**Goal**: Full-auto pipeline runs with resume capability; all session commands work; seven legacy hooks are retired atomically after pipeline is verified
**Depends on**: Phase 4
**Requirements**: SESS-01, SESS-02, SESS-03, SESS-04, SESS-05, HOOK-01, HOOK-02, HOOK-03
**Success Criteria** (what must be TRUE):
  1. Running `/seraphim:pause` during an active pipeline writes current state to `.seraphim/phases/{N}/state.json`; running `/seraphim:resume {N}` in a new session restores state and continues from where it stopped
  2. Running `/seraphim:history` displays all past pipeline runs with per-phase costs, models used, outcomes, loop counts, and total spend — verifiable by running two pipelines and confirming both appear with correct data
  3. Running `/seraphim:status` shows the active profile, current phase progress, any active overrides, and model availability for all nine models
  4. Running `/seraphim:help` displays all commands, profiles, phase descriptions, and configuration options without requiring any active pipeline
  5. After the retirement edit to `~/.claude/settings.json`, the token log contains no entries from any of the seven retired hook names during a new pipeline session; archive copies exist at a known path for rollback
**Plans:** 3 plans

Plans:
- [ ] 05-01-PLAN.md — help.md and status.md (read-only utility commands)
- [ ] 05-02-PLAN.md — history.md, pause.md, resume.md (stateful session commands)
- [ ] 05-03-PLAN.md — retire-hooks.js and atomic settings.json hook retirement

### Phase 6: Adaptive Intelligence
**Goal**: Pattern analysis produces model performance recommendations based on accumulated decisions.jsonl data; dashboard shows per-phase heatmap and profile comparison panels
**Depends on**: Phase 5
**Requirements**: ADPT-01, ADPT-02, ADPT-03, ADPT-04, ADPT-05, ADPT-06
**Success Criteria** (what must be TRUE):
  1. After enough pipeline runs accumulate, the pattern analysis engine produces a recommendation such as "Qwen Envision rejected by Judge 4/5 runs — consider Gemini 3.1 Pro" — verifiable by reading the recommendations output
  2. No recommendation is ever auto-applied — the system presents recommendations for human approval; rejected recommendations appear in the rejection log with the rejection timestamp — verifiable by rejecting a recommendation and reading the log
  3. The dashboard shows a per-phase model performance heatmap panel with success rates by (model, phase) combination — verifiable by opening dashboard.html and seeing the panel populated with data
  4. The dashboard shows a profile cost/quality comparison panel with average cost per run versus Crucible pass rate per profile — verifiable by opening dashboard.html with data from multiple profiles
**Plans**: TBD
**UI hint**: yes

### Phase 7: Multi-Project Dashboard
**Goal**: A localhost web interface aggregates progress, metrics, and workflow data across all Seraphim-managed projects into a unified command center
**Depends on**: Phase 6
**Requirements**: DASH-01, DASH-02, DASH-03, DASH-04, DASH-05, DASH-06, DASH-07
**Success Criteria** (what must be TRUE):
  1. The multi-project scanner discovers `.seraphim/` directories across `~/projects/` and any additional configured paths — verifiable by having two or more projects with `.seraphim/` and confirming both appear
  2. The dashboard at `127.0.0.1:PORT` displays a multi-project overview showing each project's name, active profile, current phase, progress bar, total cost, and last activity date
  3. Clicking a project drills down to show its phase roadmap, completed vs remaining tasks, model assignments per phase, and pipeline run history
  4. The workflow metrics panel shows cross-project model performance aggregated from all projects' decisions.jsonl, cost trends over time, and savings vs Opus-only baseline
  5. The dashboard serves from a Node.js HTTP server with no external framework dependencies — self-contained with inlined CSS/JS following the existing Chart.js dashboard pattern
**Plans**: TBD
**UI hint**: yes

## Progress

| Phase | Milestone | Plans Complete | Status | Completed |
|-------|-----------|----------------|--------|-----------|
| 1. Plugin Scaffold and Infrastructure | v3.0 | 3/3 | Complete   | 2026-04-05 |
| 2. Model Executors and Pricing | v3.0 | 5/5 | Complete   | 2026-04-05 |
| 3. Six Phase Pipeline and Profile Management | v3.0 | 6/6 | Complete   | 2026-04-08 |
| 03.1. Parallel Discovery Research Tracks | v3.0 | 0/1 | Complete    | 2026-04-08 |
| 4. Quality Gates and Decision Logging | v3.0 | 2/4 | In Progress|  |
| 5. Session Commands and Hook Consolidation | v3.0 | 0/3 | Not started | - |
| 6. Adaptive Intelligence | v3.0 | 0/? | Not started | - |
| 7. Multi-Project Dashboard | v3.0 | 0/? | Not started | - |
| 8. Thought Orphanage Integration | v3.0 | 0/? | Not started | - |
| 9. Human-AI Cognitive Division | v3.0 | 0/? | Not started | - |
| 10. Context Management and Token Optimization | v3.0 | 0/? | Not started | - |
| 11. OpenClaw Local RAG Integration | v3.0 | 0/? | Not started | - |

### Phase 8: Thought Orphanage Integration
**Goal**: A `/seraphim:thought` slash command captures seed ideas at project level (`.seraphim/thoughts/`) or global level (`~/thought-orphanage/`), with a guided braindump workflow that matures raw thoughts into structured ideas — all visible on the multi-project dashboard
**Depends on**: Phase 7
**Requirements**: TBD
**Success Criteria** (what must be TRUE):
  1. Running `/seraphim:thought` in a project directory creates a seed file under `.seraphim/thoughts/` with the braindump workflow (probe → synthesize → SEED.md)
  2. Running `/seraphim:thought --global` creates a seed in `~/thought-orphanage/seeds/` following the existing thought-orphanage conventions
  3. The multi-project dashboard shows a "Thought Orphanage" panel listing all seeds across projects with status (raw/exploring/nearly-ready), last-worked date, and graduation signals
  4. The brainstorming process guides the user from raw idea to structured SEED.md with problem, audience, vision, and graduation criteria — matching the thought-orphanage CLAUDE.md spec
**Plans**: TBD

### Phase 9: Human-AI Cognitive Division
**Goal**: Research-backed framework identifying where human cognition vs AI execution is optimally placed within Seraphim's six-phase pipeline, producing concrete routing rules and workflow recommendations
**Depends on**: Phase 3 (pipeline must exist to analyze it)
**Requirements**: TBD
**Success Criteria** (what must be TRUE):
  1. A cognitive division map exists showing each of the six pipeline phases rated on a human-leverage vs AI-execution spectrum, with evidence citations from the divergent/convergent research
  2. The framework produces actionable routing recommendations — e.g., "Envision benefits from human seeding before AI expansion" or "Judge phase should always surface decisions for human review"
  3. Research covers the Jagged Frontier, divergent/convergent boundary, and 2-hour autonomy ceiling as they apply to Seraphim's specific workflow
  4. Recommendations are integrated as configurable human-in-the-loop checkpoints in the pipeline config
**Plans**: TBD

### Phase 09.1: Human-AI Task Routing and Dashboard Views (INSERTED)

**Goal:** Wire Phase 9's Human-AI Cognitive Division research into the pipeline: Architect produces blueprints with `assignee: human` or `assignee: ai` tags on each task; Forge skips human-tagged tasks and lists them separately; the Multi-Project Dashboard (Phase 7) displays separate human and AI task lists with progress tracking for both
**Requirements**: PIPE-04
**Depends on:** Phase 7 (dashboard must exist), Phase 9 (research must exist)
**Plans:** 0 plans

Plans:
- [ ] TBD (run /gsd:plan-phase 09.1 to break down)

### Phase 10: Context Management and Token Optimization
**Goal**: Research and implement strategies to minimize token usage across the nine-model pipeline without degrading output quality, with measurable before/after cost comparisons
**Depends on**: Phase 4 (needs decisions.jsonl for measurement)
**Requirements**: TBD
**Success Criteria** (what must be TRUE):
  1. Research document covering context compression techniques, prompt optimization patterns, and caching strategies applicable to multi-model pipelines
  2. Measurable token reduction (target: 20%+ reduction in average pipeline cost) validated against baseline from decisions.jsonl
  3. Implementation of at least two concrete optimizations (e.g., context pruning between phases, prompt template compression, intelligent caching of repeated context)
  4. Dashboard panel showing token usage trends and cost savings from optimizations
**Plans**: TBD

### Phase 11: OpenClaw Local RAG Integration
**Goal**: Research OpenClaw's architecture — specifically its local RAG system for project knowledge — and adapt the pattern so Seraphim can reference project history, decisions, and documentation during pipeline phases without external dependencies
**Depends on**: Phase 3 (pipeline phases need to exist to consume RAG context)
**Requirements**: TBD
**Success Criteria** (what must be TRUE):
  1. Research document analyzing OpenClaw's RAG architecture, indexing strategy, and retrieval patterns
  2. A local RAG system indexes `.seraphim/` artifacts (decisions.jsonl, phase outputs, config) into a searchable store without requiring cloud services
  3. Pipeline phases can query project knowledge during execution — e.g., Judge can reference past decision patterns, Architect can reference prior blueprints
  4. The RAG system works offline with no external API dependencies, using local embeddings or keyword-based retrieval
**Plans**: TBD

---

## Previous Milestones

<details>
<summary>✅ v1.0 Claude X Codex (Phases 1-4) — SHIPPED 2026-04-02</summary>

- [x] Phase 1: Foundation (3/3 plans) — completed 2026-04-02
- [x] Phase 2: Review Gate & GSD Integration (2/2 plans) — completed 2026-04-02
- [x] Phase 3: Plan Review Loop & Superpowers (2/2 plans) — completed 2026-04-02
- [x] Phase 4: Cost Reporting (1/1 plan) — completed 2026-04-02

</details>

<details>
<summary>✅ v1.1 Global Metrics Dashboard (Phases 5-7) — SHIPPED 2026-04-03</summary>

- [x] Phase 5: Data Pipeline (3/3 plans) — completed 2026-04-03
- [x] Phase 6: Dashboard Generator (2/2 plans) — completed 2026-04-03
- [x] Phase 7: Charts & Hook Integration (2/2 plans) — completed 2026-04-03

</details>

<details>
<summary>✅ v2.0 Three-Model Intelligence (Phases 8-14) — SHIPPED 2026-04-03</summary>

- [x] Phase 8: MiniMax Foundation (3/3 plans) — completed 2026-04-03
- [x] Phase 9: Dual Review Gate (1/1 plan) — completed 2026-04-03
- [x] Phase 10: Adversarial Plan Review (2/2 plans) — completed 2026-04-03
- [x] Phase 11: PostToolUse Bug Scanner (1/1 plan) — completed 2026-04-03
- [x] Phase 12: Context Compression (2/2 plans) — completed 2026-04-03
- [x] Phase 13: Codex Execution Pipeline (1/1 plan) — completed 2026-04-03
- [x] Phase 14: Three-Model Reporting (2/2 plans) — completed 2026-04-03

</details>

<details>
<summary>✅ v2.0 Adaptive Intelligence — Phase 15 — SHIPPED 2026-04-04</summary>

- [x] Phase 15: Decision Capture Infrastructure (3/3 plans) — completed 2026-04-04

</details>
