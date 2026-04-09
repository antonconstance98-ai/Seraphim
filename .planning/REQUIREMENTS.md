# Requirements: Seraphim v3.1 Project Management

**Defined:** 2026-04-09
**Core Value:** Six wings, six phases, six cognitive tasks -- each assigned to the model that does it best. The human orchestrates. AI converges. Adaptive intelligence makes the system smarter over time. v3.1 adds a project management layer so the human knows **what** to build, in what order, across all projects.

## v3.1 Requirements

### Roadmap & Milestone Management

- [x] **ROAD-01**: Project-level roadmap stored in `.seraphim/roadmap.json` with milestone-to-feature hierarchy, status enum, and version tagging
- [x] **ROAD-02**: `/seraphim:roadmap` displays current roadmap as milestone-feature tree with statuses in terminal
- [x] **ROAD-03**: `/seraphim:add-feature` appends a new feature to a milestone's backlog with name, description, and priority order
- [x] **ROAD-04**: Milestone completion and archival freezes shipped milestones to `.seraphim/milestones/vX.Y.json` and cleans active roadmap
- [x] **ROAD-05**: Milestone progress percentage computed from feature statuses (complete/total) and displayed in roadmap view
- [ ] **ROAD-06**: Roadmap panel on web dashboard showing milestone-feature tree per project with visual status indicators
- [x] **ROAD-07**: Milestone cost tracking aggregates decisions.jsonl costs for all features in a milestone

### Feature Queue

- [x] **QUEUE-01**: Feature backlog with `planned` status in roadmap.json; any feature not yet started is in the backlog
- [x] **QUEUE-02**: `/seraphim:start {feature}` moves feature from `planned` to `in-progress` and launches the six-phase pipeline
- [x] **QUEUE-03**: WIP limit (configurable, default 2) enforced on `/seraphim:start`; warns if limit exceeded before starting
- [x] **QUEUE-04**: Feature reordering within a milestone via command or direct JSON edit
- [x] **QUEUE-05**: Feature dependency declarations (`depends_on` array) with start-guard check that warns if dependencies incomplete

### Human Task Management

- [x] **TASK-01**: `/seraphim:inbox` aggregates all pending human tasks across all active features and projects into a unified inbox
- [x] **TASK-02**: Human task types: decision, research, review, validation, skills -- surfaced as type labels in task lists
- [x] **TASK-03**: `/seraphim:done {task-id}` marks a human task complete without re-running the full pipeline
- [x] **TASK-04**: Pipeline gates (before Envision, before Architect, after Crucible) write human tasks to forge-log.md visible in inbox
- [x] **TASK-05**: Skills development task type with project-domain linkage and recommended resources
- [ ] **TASK-06**: Human task dashboard panel showing all tasks across projects grouped by project and type
- [x] **TASK-07**: Research task type with context injection -- on completion, research notes auto-index to project knowledge via RAG

### Cross-Project Overview

- [ ] **OVER-01**: `/seraphim:overview` shows all Seraphim projects with active milestone, features in-progress, human tasks pending, WIP count
- [ ] **OVER-02**: Dashboard cross-project panel showing all projects with PM status (milestone progress, feature counts, human tasks)
- [ ] **OVER-03**: Active-only filter (default) hides idle projects; `--all` flag shows everything
- [ ] **OVER-04**: "What needs attention" signal surfaces blocked features, exceeded WIP limits, and pending human gates prominently
- [ ] **OVER-05**: Cross-project cost trend aggregating decisions.jsonl across projects by date, rendered as trend line in dashboard

### Architecture & Integration

- [x] **ARCH-01**: PM layer is read-path only -- observes pipeline execution, never gates or blocks it; every PM operation has bypass
- [x] **ARCH-02**: `decisions-logger.js` extended with nullable `feature_id` field linking decisions to features
- [x] **ARCH-03**: `/seraphim:pause` state.json extended with PM context block (feature ID, milestone, progress) for session continuity
- [x] **ARCH-04**: Neon database extended with `milestones`, `features`, `human_tasks` tables (additive, no existing table changes)
- [ ] **ARCH-05**: Sync script extended with two new collection targets: feature_snapshots and human_task_snapshots
- [x] **ARCH-06**: Anti-features enforced: no sprint/story-points, no time-boxing, no drag-and-drop Kanban, no external PM tool sync

## v3.2+ Requirements (Deferred)

### Intelligence Layer

- **INTEL-01**: Velocity trend computation (features completed per week from phase-state timestamps)
- **INTEL-02**: ML-based urgency scoring for "what needs attention" (v3.1 uses rule-based)
- **INTEL-03**: Research task RAG handoff with automatic embedding of research outputs

### Migration

- **MIGR-01**: Bulk import from GSD ROADMAP.md format (`/seraphim:import-roadmap`)
- **MIGR-02**: Milestone cost tracking with pre-run estimates vs actual comparison

## Out of Scope

| Feature | Reason |
|---------|--------|
| Sprint/cycle system with fixed time-boxes | Solo AI-assisted work is not time-boxed; sprints add overhead for no coordination benefit |
| Story points / estimation system | Team throughput metric; meaningless for solo + AI execution |
| Drag-and-drop Kanban in browser | Features move via terminal commands; adds dependency for minimal value |
| Multi-user roadmap sharing | Solo developer context; adds auth, sync, conflict resolution complexity |
| Gantt chart with date ranges | AI build time estimates are unreliable; date-focus creates false accountability |
| Jira-style 4-level hierarchy | Milestone-Feature-Task (3 levels) is sufficient for solo developer |
| External PM sync (Notion, Linear, Jira) | Adds credentials, sync logic, data transformation for no local benefit |
| Automated task reminders/notifications | Creates notification fatigue; user checks on their own cadence |
| Time estimates per human task | Cognitive tasks are not time-predictable; adds estimation overhead |
| Project portfolio dependencies | Projects are independent execution contexts; cross-project deps resolved by human judgment |

## Traceability

| Requirement | Phase | Status |
|-------------|-------|--------|
| ROAD-01 | Phase 1 | Complete |
| ROAD-02 | Phase 1 | Complete |
| ROAD-03 | Phase 1 | Complete |
| ROAD-04 | Phase 1 | Complete |
| ROAD-05 | Phase 1 | Complete |
| ROAD-06 | Phase 3 | Pending |
| ROAD-07 | Phase 1 | Complete |
| QUEUE-01 | Phase 1 | Complete |
| QUEUE-02 | Phase 1 | Complete |
| QUEUE-03 | Phase 1 | Complete |
| QUEUE-04 | Phase 1 | Complete |
| QUEUE-05 | Phase 2 | Complete |
| TASK-01 | Phase 1 | Complete |
| TASK-02 | Phase 1 | Complete |
| TASK-03 | Phase 1 | Complete |
| TASK-04 | Phase 1 | Complete |
| TASK-05 | Phase 2 | Complete |
| TASK-06 | Phase 3 | Pending |
| TASK-07 | Phase 2 | Complete |
| OVER-01 | Phase 2 | Pending |
| OVER-02 | Phase 3 | Pending |
| OVER-03 | Phase 2 | Pending |
| OVER-04 | Phase 2 | Pending |
| OVER-05 | Phase 2 | Pending |
| ARCH-01 | Phase 1 | Complete |
| ARCH-02 | Phase 1 | Complete |
| ARCH-03 | Phase 1 | Complete |
| ARCH-04 | Phase 2 | Complete |
| ARCH-05 | Phase 2 | Pending |
| ARCH-06 | Phase 1 | Complete |

**Coverage:**
- v3.1 requirements: 30 total
- Mapped to phases: 30
- Unmapped: 0

---
*Requirements defined: 2026-04-09*
*Last updated: 2026-04-09 -- roadmap phase assignments complete*
