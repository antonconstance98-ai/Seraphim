# Roadmap: Seraphim

## Milestones

- ✅ **v1.0 Claude X Codex** — Phases 1-4 (shipped 2026-04-02) — [archive](milestones/v1.0-ROADMAP.md)
- ✅ **v1.1 Global Metrics Dashboard** — Phases 5-7 (shipped 2026-04-03) — [archive](milestones/v1.1-ROADMAP.md)
- ✅ **v2.0 Three-Model Intelligence** — Phases 8-14 (shipped 2026-04-03)
- ✅ **v2.0 Adaptive Intelligence** — Phase 15 (shipped 2026-04-04)
- ✅ **v3.0 Seraphim** — 13 phases (shipped 2026-04-09) — [archive](milestones/v3.0-ROADMAP.md)

---

## v3.1 Seraphim Project Management

**Milestone goal:** Add a project management layer so the human knows what to build, in what order, across all projects -- with roadmaps, feature queues, human task management, cross-project oversight, and dashboard panels. PM is read-path only; it never gates the pipeline.

## Phases

- [ ] **Phase 1: Core PM Primitives** - roadmap.json schema, slash commands, WIP limit, milestone archival, human task inbox, pause/resume PM context
- [ ] **Phase 2: Progress Visibility** - Feature dependencies, cross-project overview, Neon tables, sync extensions, cost trends, attention signals
- [ ] **Phase 3: Dashboard PM Panels** - Roadmap panel, human tasks panel, cross-project panel on web dashboard

## Phase Details

### Phase 1: Core PM Primitives
**Goal**: Developer can create roadmaps, queue features, start features through the pipeline, view human tasks, and close milestones -- all from terminal commands, with PM context surviving pause/resume
**Depends on**: Nothing (first phase of v3.1)
**Requirements**: ROAD-01, ROAD-02, ROAD-03, ROAD-04, ROAD-05, ROAD-07, QUEUE-01, QUEUE-02, QUEUE-03, QUEUE-04, TASK-01, TASK-02, TASK-03, TASK-04, ARCH-01, ARCH-02, ARCH-03, ARCH-06
**Success Criteria** (what must be TRUE):
  1. Running `/seraphim:roadmap` displays the current milestone-feature tree with statuses (planned, in-progress, complete) in terminal -- verifiable on any project with a roadmap.json
  2. Running `/seraphim:add-feature` appends a feature to a milestone's backlog; running `/seraphim:start {feature}` moves it to in-progress and launches the six-phase pipeline -- with WIP limit warning if limit exceeded
  3. Running `/seraphim:inbox` shows all pending human tasks (decisions, research, review, validation) across active features with type labels -- verifiable by having pipeline gates write tasks and seeing them aggregated
  4. Completing a milestone via archival freezes it to `.seraphim/milestones/vX.Y.json`, cleans active roadmap, and shows milestone cost from decisions.jsonl
  5. Running `/seraphim:pause` during a feature preserves PM context (feature ID, milestone, progress) in state.json; `/seraphim:resume` restores it -- no orphaned PM state after session restart
**Plans:** 4 plans
Plans:
- [ ] 01-01-PLAN.md — Foundation lib (roadmap.js, config/decisions-logger extensions) + roadmap display
- [ ] 01-02-PLAN.md — Feature lifecycle (add-feature, start commands)
- [ ] 01-03-PLAN.md — Human task inbox + done command
- [ ] 01-04-PLAN.md — Pause/resume PM context + milestone archival

### Phase 2: Progress Visibility
**Goal**: Cross-project oversight works from terminal and data flows into Neon for dashboard consumption -- feature dependencies are enforced, blocked features surface prominently, and cost trends aggregate across projects
**Depends on**: Phase 1
**Requirements**: QUEUE-05, TASK-05, TASK-07, OVER-01, OVER-03, OVER-04, OVER-05, ARCH-04, ARCH-05
**Success Criteria** (what must be TRUE):
  1. Running `/seraphim:overview` shows all Seraphim projects with active milestone, in-progress features, pending human tasks, and WIP count -- with active-only default and `--all` flag for idle projects
  2. Feature dependency declarations in roadmap.json trigger a start-guard warning when attempting to start a feature whose dependencies are incomplete
  3. Blocked features, exceeded WIP limits, and pending human gates surface prominently as "needs attention" signals in both overview and dashboard data
  4. Neon database has `milestones`, `features`, `human_tasks` tables populated by sync script extensions (feature_snapshots and human_task_snapshots collection targets)
  5. Cross-project cost trend data aggregates decisions.jsonl across projects by date, ready for dashboard rendering
**Plans**: TBD

### Phase 3: Dashboard PM Panels
**Goal**: The web dashboard becomes the human's PM command center with visual roadmap, human task management, and cross-project overview panels
**Depends on**: Phase 2
**Requirements**: ROAD-06, TASK-06, OVER-02
**Success Criteria** (what must be TRUE):
  1. Dashboard shows a roadmap panel per project displaying the milestone-feature tree with visual status indicators (planned/in-progress/complete)
  2. Dashboard shows a human tasks panel with all tasks across projects grouped by project and type (decision, research, review, validation, skills) with status tracking
  3. Dashboard shows a cross-project panel with PM status per project -- milestone progress, feature counts, human tasks pending, and cost trend visualization
**Plans**: TBD
**UI hint**: yes

## Progress

**Execution Order:** Phases execute in numeric order: 1 -> 2 -> 3

| Phase | Milestone | Plans Complete | Status | Completed |
|-------|-----------|----------------|--------|-----------|
| 1. Core PM Primitives | v3.1 | 0/4 | Not started | - |
| 2. Progress Visibility | v3.1 | 0/? | Not started | - |
| 3. Dashboard PM Panels | v3.1 | 0/? | Not started | - |

---

## Previous Milestones

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
