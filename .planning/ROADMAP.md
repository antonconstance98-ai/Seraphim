# Roadmap: Seraphim

## Milestones

- ✅ **v1.0 Claude X Codex** — Phases 1-4 (shipped 2026-04-02) — [archive](milestones/v1.0-ROADMAP.md)
- ✅ **v1.1 Global Metrics Dashboard** — Phases 5-7 (shipped 2026-04-03) — [archive](milestones/v1.1-ROADMAP.md)
- ✅ **v2.0 Three-Model Intelligence** — Phases 8-14 (shipped 2026-04-03)
- ✅ **v2.0 Adaptive Intelligence** — Phase 15 (shipped 2026-04-04)
- ✅ **v3.0 Seraphim** — 13 phases (shipped 2026-04-09) — [archive](milestones/v3.0-ROADMAP.md)
- ✅ **v3.1 Seraphim Project Management** — 3 phases (shipped 2026-04-09) — [archive](milestones/v3.1-ROADMAP.md)
- 🔄 **v3.2 Idea-to-Shipped Journey** — Phases 32-37 (active)

---

## v3.2 Idea-to-Shipped Journey (Active)

### Phases

- [ ] **Phase 32: Foundations** — Clear v3.1 technical debt and audit schema extensions
- [ ] **Phase 33: Core Command Layer** — Seed capture, requirements, planning, and execution commands
- [ ] **Phase 34: Research + Session + Navigation** — Research two-command system, session management, smart routing
- [ ] **Phase 35: Phase Management + Config + UI Tooling** — Phase lifecycle, configuration, and UI audit commands
- [ ] **Phase 36: Human Tasks + Debugging** — Enriched human task inbox, systematic debugging, auto-repair
- [ ] **Phase 37: Verification + Dashboard** — Goal-backward verification, UAT, and full dashboard visualization

---

## Phase Details

### Phase 32: Foundations
**Goal**: All v3.1 technical debt is cleared and every v3.2 data concept has a verified schema extension path
**Depends on**: Nothing (prerequisite gate)
**Requirements**: FOUND-01, FOUND-02, FOUND-03, FOUND-04
**Success Criteria** (what must be TRUE):
  1. Neon PM tables exist in production and queries return data without errors
  2. All PM tables use a consistent column name for the project identifier (no mixed `project`/`project_name`)
  3. `feature_id` appears in the decisions log entries written to Neon
  4. A schema extension audit doc exists confirming every v3.2 data concept maps to an existing structure
**Plans:** 3 plans
Plans:
- [x] 32-01-PLAN.md — Neon DDL migration + schema unification (project_name)
- [x] 32-02-PLAN.md — feature_id wiring through decisions pipeline
- [x] 32-03-PLAN.md — Schema extension audit document

### Phase 33: Core Command Layer
**Goal**: Users can capture ideas, define requirements, generate wave-structured plans, and execute work through native Seraphim commands
**Depends on**: Phase 32
**Requirements**: SEED-01, SEED-02, SEED-03, SEED-04, SEED-05, SEED-06, SEED-07, REQ-01, REQ-02, REQ-03, REQ-04, PLAN-01, PLAN-02, PLAN-03, PLAN-04, PLAN-05, PLAN-06, EXEC-01, EXEC-02, EXEC-03, EXEC-04, EXEC-05, EXEC-06
**Success Criteria** (what must be TRUE):
  1. User can run `/seraphim:seed` and have the idea stored in `.planning/seeds/` with a SEED-NNN.md file and index.jsonl entry
  2. User can run `/seraphim:requirements` and get AI-suggested REQ-IDs grouped by category with v1/future/out-of-scope scoping that they approve
  3. User can run `/seraphim:plan` and receive a wave-structured PLAN.md where dependencies are resolved via Kahn's algorithm before tasks appear in a wave
  4. User can run `/seraphim:execute` and have all plans in a phase execute with wave-based parallelism (independent tasks run together, dependents wait)
  5. User can run `/seraphim:discuss` before planning and get a CONTEXT.md with locked implementation decisions
**Plans:** 5 plans
Plans:
- [x] 33-01-PLAN.md — Core lib modules (seed-store.js, requirements.js, wave-planner.js)
- [x] 33-02-PLAN.md — Seed/note/todo commands (seed, note, add-todo, check-todos, promote)
- [x] 33-03-PLAN.md — Requirements + discuss + assumptions commands
- [x] 33-04-PLAN.md — Plan command with wave resolution and checker loop
- [x] 33-05-PLAN.md — Execution commands (execute, execute-plan, autonomous, quick, fast)
**UI hint**: yes

### Phase 34: Research + Session + Navigation
**Goal**: Users can scope and run two-command research, pause and resume sessions with full context, and navigate to the next logical action automatically
**Depends on**: Phase 33
**Requirements**: RSRCH-01, RSRCH-02, RSRCH-03, RSRCH-04, RSRCH-05, SESS-01, SESS-02, SESS-03, NAV-01, NAV-02, NAV-03
**Success Criteria** (what must be TRUE):
  1. User cannot run `/seraphim:research-run` without first having run `/seraphim:research-scope` — the interrogation gate is enforced
  2. User can run `/seraphim:pause` and find a HANDOFF.json and `.continue-here.md` that restore full working context on next session
  3. User can run `/seraphim:next` and be routed to the correct next command (discuss→plan→execute→verify) based on current project state
  4. User can run `/seraphim:map-codebase` and receive a structured codebase map produced by parallel mapper agents
**Plans:** 4 plans
Plans:
- [x] 34-01-PLAN.md — Research tracker lib + scope/run commands
- [x] 34-02-PLAN.md — Session management (pause/resume/report)
- [x] 34-03-PLAN.md — Navigation routing (next/do/progress)
- [x] 34-04-PLAN.md — Codebase mapping with parallel agents

### Phase 35: Phase Management + Config + UI Tooling
**Goal**: Users can manage the full phase lifecycle from interactive commands, configure workflow settings, and run UI audits and test generation
**Depends on**: Phase 33
**Requirements**: MGMT-01, MGMT-02, MGMT-03, MGMT-04, MGMT-05, MGMT-06, MGMT-07, MGMT-08, CFG-01, CFG-02, UI-01, UI-02, UI-03
**Success Criteria** (what must be TRUE):
  1. User can add, insert, remove, and reorder phases through `/seraphim:add-phase`, `/seraphim:insert-phase`, and `/seraphim:remove-phase` without orphaning requirements
  2. User can complete a milestone via `/seraphim:complete-milestone` and get an archived snapshot with git tag and cost attribution
  3. User can run `/seraphim:ui-spec` for a frontend phase and receive a design contract that specifies layout, components, and interaction patterns
  4. User can run `/seraphim:settings` and change model profile or workflow toggles that persist to config.json
**Plans:** 4 plans
Plans:
- [x] 35-01-PLAN.md — ROADMAP.md manipulation lib + add/insert/remove-phase commands
- [x] 35-02-PLAN.md — Milestone lifecycle (complete-milestone, pr-branch, health)
- [x] 35-03-PLAN.md — Workstreams, manager, and settings commands
- [x] 35-04-PLAN.md — UI and quality tooling (ui-spec, ui-review, add-tests)
**UI hint**: yes

### Phase 36: Human Tasks + Debugging
**Goal**: Human task inbox items carry skills-to-learn, thought-prompt, and research-task fields, and users have systematic debug and forensic commands with auto-repair
**Depends on**: Phase 33
**Requirements**: HTASK-01, HTASK-02, HTASK-03, DBG-01, DBG-02, DBG-03, DBG-04
**Success Criteria** (what must be TRUE):
  1. Every human task in the inbox has optional `skills_to_learn`, `thought_prompt`, and `research_task` fields that Claude populates when relevant
  2. User can run `/seraphim:debug` across session resets and have persistent debug state that accumulates findings without repeating prior investigation
  3. User can run `/seraphim:forensics` and receive a read-only root-cause report without any side effects to the codebase
  4. A failed task triggers automatic RETRY/DECOMPOSE/PRUNE/ESCALATE repair within the configured budget before surfacing to human
**Plans:** 3 plans
Plans:
- [x] 36-01-PLAN.md — Human task enrichment (migration, push-client, ingest, inbox)
- [x] 36-02-PLAN.md — Debug/forensics commands and repair.js strategy cascade
- [x] 36-03-PLAN.md — Pipeline enrichment wiring and auto-repair integration

### Phase 37: Verification + Dashboard
**Goal**: Users can verify built features against requirements with goal-backward traceability, run UAT, and see full progress visualization in the dashboard
**Depends on**: Phase 36, Phase 33
**Requirements**: VFY-01, VFY-02, VFY-03, VFY-04, VFY-05, VFY-06, VIZ-01, VIZ-02, VIZ-03, VIZ-04, VIZ-05
**Success Criteria** (what must be TRUE):
  1. User can run `/seraphim:verify` and receive a report that traces every built feature back to its REQ-ID, with at least one REQUIRES_HUMAN_JUDGMENT item
  2. User can run `/seraphim:uat` conversationally and have UAT.md accumulate findings with persistent state across resets
  3. Dashboard shows a progress bar and completion percentage for each phase and the overall milestone
  4. Dashboard shows a rolling 7-day velocity chart and a full roadmap tree with phases, waves, tasks, costs, and metrics
**Plans:** 4 plans
Plans:
- [ ] 34-01-PLAN.md — Research tracker lib + scope/run commands
- [ ] 34-02-PLAN.md — Session management (pause/resume/report)
- [ ] 34-03-PLAN.md — Navigation routing (next/do/progress)
- [x] 34-04-PLAN.md — Codebase mapping with parallel agents
**UI hint**: yes

---

## Progress Table

| Phase | Plans Complete | Status | Completed |
|-------|----------------|--------|-----------|
| 32. Foundations | 3/3 | Complete    | 2026-04-10 |
| 33. Core Command Layer | 5/5 | Complete    | 2026-04-10 |
| 34. Research + Session + Navigation | 4/4 | Complete    | 2026-04-10 |
| 35. Phase Management + Config + UI Tooling | 4/4 | Complete    | 2026-04-10 |
| 36. Human Tasks + Debugging | 3/3 | Complete   | 2026-04-10 |
| 37. Verification + Dashboard | 0/? | Not started | - |

---

## Previous Milestones

<details>
<summary>v3.1 Seraphim Project Management (3 phases) -- SHIPPED 2026-04-09</summary>

See [archive](milestones/v3.1-ROADMAP.md) for full phase details.

</details>

<details>
<summary>v3.0 Seraphim (13 phases) -- SHIPPED 2026-04-09</summary>

See [archive](milestones/v3.0-ROADMAP.md) for full phase details.

</details>

<details>
<summary>v1.0 Claude X Codex (Phases 1-4) -- SHIPPED 2026-04-02</summary>

See [archive](milestones/v1.0-ROADMAP.md) for full phase details.

</details>

<details>
<summary>v1.1 Global Metrics Dashboard (Phases 5-7) -- SHIPPED 2026-04-03</summary>

See [archive](milestones/v1.1-ROADMAP.md) for full phase details.

</details>

<details>
<summary>v2.0 Three-Model Intelligence (Phases 8-15) -- SHIPPED 2026-04-04</summary>

Phases 8-14: Three-Model Intelligence. Phase 15: Adaptive Intelligence.

</details>
