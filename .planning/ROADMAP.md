# Roadmap: Seraphim

## Milestones

- ✅ **v1.0 Claude X Codex** — Phases 1-4 (shipped 2026-04-02) — [archive](milestones/v1.0-ROADMAP.md)
- ✅ **v1.1 Global Metrics Dashboard** — Phases 5-7 (shipped 2026-04-03) — [archive](milestones/v1.1-ROADMAP.md)
- ✅ **v2.0 Three-Model Intelligence** — Phases 8-14 (shipped 2026-04-03)
- ✅ **v2.0 Adaptive Intelligence** — Phase 15 (shipped 2026-04-04)
- 🔄 **v3.0 Seraphim** — Phases 1-7 (in progress) — clean break, phase numbering reset

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
**Plans**: TBD
**Research flag**: Gemini SDK search grounding + thinking mode APIs and Perplexity MCP bridge pattern need research before planning this phase

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
**Plans**: TBD
**UI hint**: yes

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
**Plans**: TBD

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
**Plans**: TBD

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
| 2. Model Executors and Pricing | v3.0 | 0/? | Not started | - |
| 3. Six Phase Pipeline and Profile Management | v3.0 | 0/? | Not started | - |
| 4. Quality Gates and Decision Logging | v3.0 | 0/? | Not started | - |
| 5. Session Commands and Hook Consolidation | v3.0 | 0/? | Not started | - |
| 6. Adaptive Intelligence | v3.0 | 0/? | Not started | - |
| 7. Multi-Project Dashboard | v3.0 | 0/? | Not started | - |

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
