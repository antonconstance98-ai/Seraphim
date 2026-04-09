# Feature Landscape: Seraphim v3.1 Project Management

**Domain:** Project management layer for a solo-developer AI-assisted creative pipeline
**Researched:** 2026-04-08
**Confidence:** HIGH — grounded in GSD source analysis, Seraphim existing commands, and PM ecosystem research (Linear, GitHub Projects, Jira, Kanban, Agile literature)

---

## Context: Subsequent Milestone Scope

v3.0 ships a six-phase execution pipeline — it takes a single feature from idea to verified output.
v3.1 adds a **project management layer above the pipeline** — managing what gets fed into the pipeline, when, and in what order.

The distinction is critical:

| Layer | What It Manages | v3.0 Status |
|-------|----------------|-------------|
| Pipeline (Discover → Crucible) | How one feature is built | Shipped |
| Project management | What to build, in what order, across all projects | This milestone |

Every feature below is additive. Nothing in v3.0 is being replaced.

---

## Existing Capabilities (Do Not Rebuild)

The following are already in place and feed the PM layer:

| Existing Component | What It Provides |
|-------------------|-----------------|
| `/seraphim:run {phase-id}` | Executes the six-phase pipeline for a single feature |
| `phase-state.js` | Tracks per-feature pipeline progress (complete/in-progress/not-started) |
| `decisions.jsonl` | Per-phase cost, outcome, loop counts — raw data for PM reporting |
| `token-log.jsonl` | Per-call cost tracking across all nine models |
| `/seraphim:tasks {phase-id}` | Lists human vs AI tasks for a given feature, with status from blueprint.md + forge-log.md |
| Seraphim web dashboard (v3.0) | Multi-project metrics, session history, per-phase heatmap — already deployed |
| `multi-project-scanner.js` | Scans all projects for Seraphim state — foundation for cross-project overview |

---

## Feature Landscape

### 1. Project Roadmap and Milestone Management

**What GSD does here:** GSD maintains `ROADMAP.md` (milestone-scoped phase list), `STATE.md` (current progress), and `.planning/milestones/` archives. Milestones are version-tagged (v1.0, v2.0). Phase numbers are sequential integers or decimals for inserted phases. GSD's `roadmap analyze` command parses all phases and returns structured JSON with completion status per phase.

**What Seraphim lacks:** GSD's planning artifacts live in `.planning/` and are consumed only by GSD commands. Seraphim has no equivalent concept of a milestone, roadmap, or version above the individual feature (phase-id).

#### Table Stakes

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| Project-level roadmap (ordered feature list with milestones) | Without a roadmap, there is no plan — just a pile of phase-ids with no ordering or grouping | LOW | Stored in `.seraphim/roadmap.json`: milestones array, each with features array and target version. Simple JSON; no database. |
| Milestone concept (version-tagged groups of features) | Releases need scope definition; solo developers need to know "what ships in v3.1" vs "what ships in v4.0" | LOW | Milestone = `{id, version, name, status: planned/active/complete, features: [phase-id, ...]}` |
| Feature status tracking (planned / in-progress / complete / blocked) | Without status, roadmap is static text; useless for daily decision-making | LOW | Derived from `phase-state.js` + a thin status layer; status updates when pipeline phases complete |
| `/seraphim:roadmap` command to view current roadmap | Core PM visibility; must be terminal-first since the user manages everything from CLI | LOW | Reads roadmap.json + phase-state data; prints milestone → feature tree with statuses |
| Milestone completion and archival | Shipped milestones should be frozen; active roadmap should stay clean | LOW | `complete-milestone` marks all features done, writes archive to `.seraphim/milestones/vX.Y.json` |

#### Differentiators

| Feature | Value Proposition | Complexity | Notes |
|---------|-------------------|------------|-------|
| Roadmap view on web dashboard | Cross-project roadmaps visible at a glance; better than reading JSON files | MEDIUM | Extend existing dashboard with roadmap panel per project; render milestone → feature tree as visual kanban-style columns |
| Milestone cost tracking (actual vs estimated) | Show how much the current milestone actually cost to build; unique to AI-assisted workflow PM | MEDIUM | Sum decisions.jsonl costs for all features in milestone; compare against per-feature pre-run estimates if available |
| Progress percentage per milestone | Instant progress signal without reading individual feature statuses | LOW | Computed field: complete_features / total_features × 100; displayed in `/seraphim:roadmap` output |

#### Anti-Features

| Feature | Why Avoid | What to Do Instead |
|---------|-----------|-------------------|
| Multi-user roadmap sharing / collaborative editing | Solo developer context; adds auth, sync, conflict resolution complexity | JSON files in git; share via repo |
| Gantt chart with date ranges | Estimates for AI-assisted builds are unreliable; date-focus creates false accountability | Focus on sequence and milestone scope, not calendar dates |
| Jira-style epic hierarchy (epic → story → task → sub-task) | Four levels of nesting for a solo developer creates bureaucracy | Milestone → Feature (phase-id) → Task (blueprint markers) — three levels maximum |

---

### 2. Feature Queue

**What GSD does here:** GSD has no feature queue concept — phases are pre-defined in ROADMAP.md before the milestone starts. There is no mechanism to add features mid-milestone, prioritize them, or feed them into the pipeline on demand.

**What existing PM systems do:** Linear's "backlog" and GitHub Projects' "no status" column serve as the holding area for planned-but-not-started work. Linear uses cycles (sprints) to pull features from backlog into active work. Kanban uses a WIP limit to control how many items are in-progress simultaneously.

#### Table Stakes

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| Feature backlog (planned items not yet started) | Core PM primitive; without a backlog there is no queue to manage | LOW | Status `planned` in roadmap.json; any feature not yet run through pipeline |
| Add feature to queue (`/seraphim:add-feature`) | Features emerge throughout a project; no mechanism to capture them is a fatal gap | LOW | Appends to roadmap.json with status `planned`; prompts for feature name, milestone assignment, priority order |
| Reorder features in queue | Priorities shift; queue must be mutable | LOW | Slash command or direct JSON edit; reorder features array within milestone |
| WIP limit (max N features in-progress simultaneously) | Without WIP limit, AI-assisted context fragmentation is a real risk; too many in-flight features degrades quality per feature | LOW | Config option: `max_wip: 2` (default); `/seraphim:run` warns if WIP limit exceeded before starting |
| Feature → pipeline handoff (`/seraphim:start {feature}`) | The bridge between PM layer and execution layer; must be explicit, not automatic | LOW | Moves feature status from `planned` to `in-progress`, then invokes `/seraphim:run` |

#### Differentiators

| Feature | Value Proposition | Complexity | Notes |
|---------|-------------------|------------|-------|
| Queue view filtered by milestone | Show only features for the current milestone; avoid noise from future work | LOW | Filter param to `/seraphim:roadmap --milestone vX.Y` |
| Feature dependency declaration (`depends_on: [phase-id]`) | Some features cannot start until another completes; making dependencies explicit prevents out-of-order execution | LOW | `depends_on` array in feature JSON; `/seraphim:start` warns if dependency not complete |
| Bulk import from existing ROADMAP.md | Migration path from GSD-style planning; allows bootstrapping queue from existing documents | MEDIUM | Parse GSD ROADMAP.md format into roadmap.json; command: `/seraphim:import-roadmap` |

#### Anti-Features

| Feature | Why Avoid | What to Do Instead |
|---------|-----------|-------------------|
| Automatic prioritization via AI scoring | Removes human agency from what to work on next — the core strategic decision | Human reorders queue manually; AI provides cost/complexity estimates as inputs to human decision |
| Sprint/cycle system with fixed time-boxes | Solo AI-assisted development is not time-boxed; sprints create overhead for no coordination benefit | Use milestone-scoped queues instead; milestones have a scope, not a deadline |
| Story points / estimation system | Velocity tracking is a team throughput metric; meaningless for solo + AI execution | Track actual cost and time from decisions.jsonl; no pre-estimation theater |

---

### 3. Progress Tracking

**What GSD does here:** GSD's `STATE.md` tracks completed phases, current phase, blockers, and decisions. `roadmap analyze` returns completion status per phase with `disk_status` (has output files) and `roadmap_complete` (marked complete in ROADMAP.md). Progress is binary per phase — complete or not.

**What Seraphim has:** `phase-state.js` tracks pipeline phase completion per feature. `/seraphim:status` shows config but not cross-feature progress. `/seraphim:tasks` shows task-level status for one feature.

#### Table Stakes

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| Per-feature pipeline progress (which of 6 phases complete) | Users need to know where a feature is in the pipeline without reading output files | LOW | Already partially exists via `phase-state.js`; needs surface in PM commands |
| Cross-project progress overview | Managing multiple projects from CLI requires a single command that shows all project statuses | MEDIUM | Extend `multi-project-scanner.js` to include PM layer data (milestone, features in-progress, WIP count) |
| Blocked feature flagging | Blocked features need explicit visibility; they silently block milestone completion if untracked | LOW | Status `blocked` in roadmap.json with required `blocked_reason` field; surface prominently in `/seraphim:roadmap` output |
| Milestone completion percentage | Instant health signal at milestone level | LOW | Computed from feature statuses; displayed in roadmap view and dashboard |

#### Differentiators

| Feature | Value Proposition | Complexity | Notes |
|---------|-------------------|------------|-------|
| Cost-to-date per feature and per milestone | AI-assisted PM is unique in having granular, real cost data per feature; expose it | LOW | Sum decisions.jsonl for phase-id; group by milestone; display in roadmap view |
| Phase-level progress breakdown in dashboard | Visualize across all in-progress features: which are at Discover, which at Forge, which at Crucible | MEDIUM | Extend dashboard with per-feature pipeline position; data from phase-state.js aggregated by multi-project-scanner.js |
| Velocity trend (features completed per week) | Solo developer equivalent of team throughput; useful for realistic milestone scoping | MEDIUM | Computed from timestamps on phase-state completion records; rendered as trend line in dashboard |

#### Anti-Features

| Feature | Why Avoid | What to Do Instead |
|---------|-----------|-------------------|
| Burndown charts | Requires upfront story point estimates which this system explicitly avoids | Use feature completion rate and cost-to-date instead |
| Time-tracking per feature (wall clock hours) | Misleading for AI-assisted work where most execution happens autonomously | Track AI cost (from decisions.jsonl) and human decision count (from gate interactions) as proxies for effort |

---

### 4. Human Task Management

**What GSD does here:** GSD has no explicit human task concept separate from AI tasks. Everything is a "plan" with "tasks". The human is expected to read plans and decide what to execute.

**What Seraphim v3.0 has:** `/seraphim:tasks` distinguishes `assignee: human` vs `assignee: ai` task markers in blueprint.md. Phase 9 (human-AI cognitive division) adds three mandatory human gates: before Envision, before Architect, after Crucible. Human tasks are captured in forge-log.md under `SERAPHIM:HUMAN_TASKS`.

**The gap:** Human tasks are currently scoped to one feature's blueprint. There is no PM-level view of all human tasks across all features and projects, no research task type, no skills development task type.

#### Table Stakes

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| Human task inbox (all pending human tasks across projects) | Without aggregation, human tasks are buried in individual forge-log.md files | MEDIUM | Scan `SERAPHIM:HUMAN_TASKS` across all active features; render unified inbox in terminal |
| Human task types: decision, research, review, validation, skills | Different human task types have different urgency and cognitive load; conflating them obscures what actually needs attention | LOW | Enum in task marker schema; surface type in task lists |
| Human task completion workflow | Human must be able to mark a task complete without re-running the full pipeline | LOW | `/seraphim:done {task-id}` marks the task complete in forge-log.md; `/seraphim:forge {phase-id}` continues with remaining AI tasks |
| Pipeline gates as human tasks | Phase 9's mandatory human checkpoints (before Envision, before Architect, after Crucible) should appear as human tasks in the inbox, not just as pipeline prompts | LOW | Gate events write a human task to forge-log.md; visible in task inbox alongside blueprint tasks |
| Skills development task type with project-domain linkage | Phase 9's D-09/D-10 design decision: skill recommendations tied to project content; these need a task type that survives pipeline runs | LOW | `type: skills` task with `domain` field (e.g., "persuasion frameworks"); displayed in inbox with recommended resources |

#### Differentiators

| Feature | Value Proposition | Complexity | Notes |
|---------|-------------------|------------|-------|
| Human task dashboard panel | Visual inbox for all human tasks across all projects; captures the "what do I need to do today" question | MEDIUM | Extend Phase 7 web dashboard with human tasks panel grouped by project and type |
| Research task type with context injection | Research tasks produce outputs (notes, references) that feed back into the pipeline via RAG indexing | MEDIUM | `type: research` task; on completion, prompt for research notes; auto-index to project knowledge via existing RAG tools |
| Task urgency signals based on pipeline position | A human decision task blocking Architect is more urgent than a skills task; surface urgency automatically | LOW | Urgency = pipeline position of the blocked feature; sort inbox by blocked feature pipeline position |

#### Anti-Features

| Feature | Why Avoid | What to Do Instead |
|---------|-----------|-------------------|
| Time estimates per human task | Cognitive tasks are not time-predictable; adds estimation overhead for no scheduling benefit | Let human pull tasks when ready; urgency signal is enough |
| Automated human task reminders / notifications | Creates notification fatigue; user checks dashboard and `/seraphim:tasks` on their own cadence | Dashboard panel visible whenever user checks status; no push notifications |
| Complex human task workflows (sub-tasks, dependencies, assignments) | Solo context; any team PM feature here adds overhead | Flat list with type, urgency, and blocked-feature linkage is sufficient |

---

### 5. Cross-Project Overview

**What exists:** `multi-project-scanner.js` already scans `~/projects/` for Seraphim state. Phase 7 dashboard renders per-project metrics. The scanner returns project list with token costs, session counts, and last activity.

**The gap:** Scanner returns execution metrics. It does not return PM-layer data: milestone progress, feature queue depth, WIP count, blocked features, or human tasks pending per project.

#### Table Stakes

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| Cross-project status summary in terminal | Primary workflow entry point; `seraphim:overview` shows all projects in one glance | LOW | Extend scanner output to include: active milestone, features in-progress, human tasks pending, WIP count |
| Dashboard cross-project panel | Visual complement to terminal command; both views of the same data | MEDIUM | Extend existing dashboard with a summary table/cards showing all projects with PM status |
| Filter to active projects only | Most projects are idle; default view should show only what needs attention | LOW | Filter: projects with status `active` or with human tasks pending; `--all` flag to show everything |

#### Differentiators

| Feature | Value Proposition | Complexity | Notes |
|---------|-------------------|------------|-------|
| Cross-project cost trend | See total AI spend across all projects over time; helps with $15/day budget management | MEDIUM | Aggregate decisions.jsonl across projects by date; render as multi-project cost trend line in dashboard |
| "What needs attention" signal | Surfaces the one or two projects that are blocked, over WIP limit, or have pending human gates | LOW | Computed: blocked features + exceeded WIP + pending human tasks with gate type; display prominently in overview |

#### Anti-Features

| Feature | Why Avoid | What to Do Instead |
|---------|-----------|-------------------|
| Project portfolio management (dependencies between projects) | Overcomplication for solo developer; projects are independent execution contexts | Track project status; cross-project dependencies are resolved by human judgment |
| External sync (Notion, Linear, Jira) | Adds credentials, sync logic, and data transformation for no local benefit | All PM state in `.seraphim/` JSON files; share via git if needed |

---

## Feature Dependencies

```
roadmap.json schema
    └──required by──> /seraphim:roadmap command
    └──required by──> /seraphim:add-feature command
    └──required by──> Milestone completion and archival
    └──required by──> Feature queue (planned/in-progress/blocked)
    └──required by──> Progress tracking (per-feature and per-milestone)
    └──required by──> Cross-project scanner extension

phase-state.js (already exists, v3.0)
    └──feeds──> Feature status in roadmap.json
    └──feeds──> Progress tracking commands

decisions.jsonl (already exists, v3.0)
    └──feeds──> Cost-to-date per feature and milestone
    └──feeds──> Velocity trend
    └──feeds──> Cross-project cost trend

multi-project-scanner.js (already exists, v3.0)
    └──extended by──> Cross-project PM overview
    └──extended by──> Dashboard PM panels

SERAPHIM:HUMAN_TASKS in forge-log.md (already exists, v3.0 via /seraphim:tasks)
    └──extended by──> Human task inbox aggregation
    └──extended by──> Dashboard human tasks panel

Phase 9 human gate decisions (context from 09-CONTEXT.md)
    └──required by──> Pipeline gate → human task writing
    └──required by──> Human task types (D-01: three mandatory gates)
    └──required by──> Skills development tasks (D-09, D-10)

Web dashboard (already exists, v3.0)
    └──extended by──> Roadmap panel
    └──extended by──> Human tasks panel
    └──extended by──> Cross-project PM summary panel
    └──extended by──> Phase-level progress visualization
```

### Dependency Notes

- **roadmap.json is the foundational new artifact.** All PM features read from or write to it. Its schema must be designed first and locked before any PM commands are built.
- **All new features extend existing v3.0 infrastructure.** Nothing requires new storage systems, databases, or external services. PM state is JSON files in `.seraphim/`.
- **Human task inbox depends on a consistent task marker schema across pipeline phases.** Phase 9's gate decisions (D-01 through D-03) define when gates fire; the PM layer depends on those gates writing to the same schema as blueprint.md human task markers.
- **Dashboard extensions are all read-only.** The dashboard reads existing JSON; it does not write state. Dashboard panels can be built independently once their data sources exist.
- **WIP limit enforcement needs roadmap.json before it can read in-progress features.** WIP check in `/seraphim:run` is gated on roadmap.json existing in the project.

---

## MVP Definition

### Launch With (Phase 1: Core PM primitives)

Minimum for the PM layer to be usable — without these, there is no project management system.

- [ ] `roadmap.json` schema definition (milestone → feature hierarchy, status enum, depends_on)
- [ ] `/seraphim:roadmap` — view current roadmap (milestone → feature tree with statuses)
- [ ] `/seraphim:add-feature` — add a feature to the backlog
- [ ] `/seraphim:start {feature}` — move feature from planned to in-progress, launch pipeline
- [ ] WIP limit check in `/seraphim:run` — warn if max_wip exceeded
- [ ] Milestone completion and archival (`/seraphim:complete-milestone`)
- [ ] Human task inbox (`/seraphim:inbox`) — aggregate SERAPHIM:HUMAN_TASKS from all active features

### Add After Core PM Works (Phase 2: Progress visibility)

- [ ] Cost-to-date per feature and per milestone (from decisions.jsonl)
- [ ] Cross-project overview (`/seraphim:overview`) — extend multi-project-scanner.js
- [ ] Roadmap panel in web dashboard
- [ ] Human tasks panel in web dashboard
- [ ] Feature dependency declarations and start-guard check
- [ ] Blocked feature flagging with reason

### Add After Visibility Works (Phase 3: Intelligence)

- [ ] Velocity trend computation (features/week from phase-state timestamps)
- [ ] Cross-project cost trend in dashboard
- [ ] "What needs attention" signal in overview
- [ ] Research task type with RAG index handoff
- [ ] Bulk import from GSD ROADMAP.md format (`/seraphim:import-roadmap`)
- [ ] Milestone cost tracking (actual vs estimated in dashboard)

---

## PM System Reference: What Works for Solo AI-Assisted Development

Drawing from GSD, Linear, GitHub Projects, Kanban, and Agile — filtered for solo developer + AI execution context:

### From GSD (adopt directly)

- **Milestone → phase hierarchy** maps cleanly to milestone → feature in Seraphim PM
- **STATE.md blocker section** → blocked status in roadmap.json with required reason
- **`roadmap analyze` JSON output pattern** → same pattern for seraphim roadmap commands
- **Phase archive on completion** → milestone archive to `.seraphim/milestones/`
- **Decimal phase insertion (5.1)** → feature can be inserted into a milestone mid-execution without renumbering

### From Linear (adopt selectively)

- **"Cycles" concept** → not needed (no time-boxing for solo AI work)
- **First-class keyboard-driven UX** → terminal-first commands follow this principle already
- **Issue status as a workflow state, not just done/not-done** → adopt: planned / in-progress / blocked / complete
- **Project-level roadmap separate from issue tracker** → adopt: roadmap.json separate from pipeline phases

### From Kanban (adopt selectively)

- **WIP limits** → adopt: `max_wip` config option; warn before exceeding
- **Visual board** → adopt for dashboard only; not for terminal (terminal uses tree/list views)
- **Continuous flow over sprints** → adopt: no sprint/cycle concept in Seraphim PM

### From Agile / Jira (reject most)

- **Story points** → reject: meaningless for AI-assisted execution
- **Sprint ceremonies (standup, retrospective, planning poker)** → reject: team coordination overhead
- **Epic hierarchy below milestone** → reject: milestone → feature → task is enough
- **Velocity / burndown** → partial adoption: track feature completion rate, not story points

### From GitHub Projects (adopt selectively)

- **Board, table, roadmap views** → adopt for dashboard; terminal shows list/tree
- **Filter by milestone, status** → adopt: `/seraphim:roadmap --milestone` filter flag
- **No-status column as backlog** → equivalent to `planned` status in roadmap.json

---

## Sources

- GSD source analysis: `~/.claude/get-shit-done/workflows/autonomous.md`, `plan-phase.md`
- Seraphim existing commands: `~/.claude/plugins/seraphim/commands/tasks.md`, `run.md`
- Phase 9 human-AI context: `.planning/phases/09-human-ai-cognitive-division/09-CONTEXT.md`
- [Linear app review — features for solo developers](https://www.morgen.so/blog-posts/linear-project-management)
- [Linear vs GitHub Issues comparison](https://cotera.co/articles/linear-vs-github-issues-comparison)
- [Kanban vs Agile comparison 2025](https://project-management.com/kanban-vs-agile/)
- [GitHub Projects documentation](https://docs.github.com/en/issues/planning-and-tracking-with-projects/learning-about-projects/about-projects)
- Project context: `.planning/PROJECT.md`

---

*Feature research for: Seraphim v3.1 Project Management layer*
*Researched: 2026-04-08*
