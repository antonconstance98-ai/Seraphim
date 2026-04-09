# Project Research Summary

**Project:** Seraphim v3.1 — Project Management Layer
**Domain:** Project management layer atop an AI-assisted multi-model execution pipeline (solo developer)
**Researched:** 2026-04-08
**Confidence:** HIGH

## Executive Summary

Seraphim v3.1 adds a project management layer above the v3.0 six-phase pipeline. The core insight from all four research streams is that this requires almost no new infrastructure. The existing stack (Node.js, JSONL, Neon Postgres, Next.js dashboard) handles everything. Zero new npm packages are needed. The PM layer is fundamentally a set of new file schemas (roadmap.json, feature-queue.jsonl, human-tasks.md), new slash commands that read/write them, two new Neon tables for cross-project dashboard views, and dashboard panels that render the data.

The recommended approach follows a single foundational principle: **PM is a read-path, not a write-path**. The PM layer observes pipeline execution; it never gates or blocks it. This principle, surfaced independently by both FEATURES.md (anti-features rejecting sprint/story-point patterns) and PITFALLS.md (ceremony creep as critical threat), is the most important architectural constraint for v3.1. The roadmap.json file is the foundational new artifact — every PM feature depends on its schema. Design it first, lock it, then build everything else on top.

The primary risk is second system effect: v3.0 shipped successfully, generating confidence that can inflate v3.1 scope into a standalone PM tool rather than a thin PM feature. The mitigation is a hard feature cap, explicit anti-features list, and the three-phase structure identified by FEATURES.md (core primitives, then progress visibility, then intelligence). Each phase delivers a usable increment. Nothing in v3.0 is replaced; everything is additive.

## Key Findings

### Recommended Stack

No new packages. The entire PM layer builds on existing infrastructure with new file formats and two Neon tables.

**Core additions (delta only):**
- `roadmap.json` per project: milestone-to-feature hierarchy, status enum, dependency declarations — the single source of PM intent
- `feature-queue.jsonl` per project: append-only feature event log, consistent with existing decisions.jsonl and token-log.jsonl patterns
- `human-tasks.md` per project: GFM task list with frontmatter, human-readable and human-editable, consistent with GSD STATE.md pattern
- `feature_snapshots` + `human_task_snapshots` Neon tables: dashboard read projections synced by the existing sync script (two new collection targets, same pattern)
- Dashboard panels: Next.js Server Components reading from Neon, no client-side state management, no drag-and-drop libraries

**Explicitly rejected:** SQLite, Drizzle/Prisma, react-beautiful-dnd, react-query, GitHub Projects API, Jira/Linear integration, dedicated PM database, full-text search engine.

### Expected Features

**Must have (table stakes):**
- roadmap.json schema with milestone-to-feature hierarchy and status tracking
- `/seraphim:roadmap` — view current roadmap as milestone-feature tree with statuses
- `/seraphim:add-feature` — capture features as they emerge
- `/seraphim:start {feature}` — bridge from PM layer to pipeline execution
- WIP limit enforcement (default max 2 in-progress features)
- Milestone completion and archival with carry-forward of incomplete items
- Human task inbox aggregating SERAPHIM:HUMAN_TASKS across all active features
- Per-feature pipeline progress (which of 6 phases complete)
- Blocked feature flagging with required reason

**Should have (differentiators):**
- Cost-to-date per feature and per milestone from decisions.jsonl
- Cross-project overview extending multi-project-scanner.js
- Roadmap and human task panels on web dashboard
- Feature dependency declarations with start-guard checks
- Milestone progress percentage (computed field)

**Defer (v2+):**
- Velocity trend computation
- Cross-project cost trend in dashboard
- "What needs attention" signal
- Research task type with RAG index handoff
- Bulk import from GSD ROADMAP.md format
- Milestone cost tracking (actual vs estimated)

### Architecture Approach

v3.1 integrates into the existing v3.0 plugin architecture without modifying any v3.0 components. The data flow is: slash commands write to `.seraphim/` files (roadmap.json, feature-queue.jsonl, human-tasks.md), the existing sync script projects this data into Neon tables, and dashboard Server Components render from Neon. The pipeline feeds PM state implicitly — phase-state.js completion events update feature status in roadmap.json. The PM layer never writes to pipeline state files; it only reads them.

**Major components:**
1. `roadmap.json` schema and read/write library — foundational data model for all PM features
2. PM slash commands (roadmap, add-feature, start, inbox, overview, complete-milestone) — terminal-first interface
3. Sync script extensions — two new collection targets (feature_snapshots, human_task_snapshots) added to existing sync pattern
4. Dashboard PM panels — read-only Kanban, roadmap tree, human tasks, cross-project summary

### Critical Pitfalls

1. **Second system effect** — v3.1 scope inflates into a standalone PM tool. Prevent by defining anti-features before building, capping feature count, and requiring every feature to answer "does this make a pipeline run faster or better?"
2. **Ceremony creep** — PM layer becomes a prerequisite tree that blocks pipeline execution. Prevent by making PM a read-path only; pipeline commands never depend on PM state being valid. Every PM operation has `--quick` or `--no-pm` bypass.
3. **File-based state inconsistency across projects** — cross-project PM reads join data from multiple file locations without freshness checks. Prevent by treating per-project `.seraphim/` as source of truth, using append-only JSONL for global state, and write-then-rename for atomic file operations.
4. **Session continuity gap** — PM state orphaned when pipeline pauses mid-feature. Prevent by extending `/seraphim:pause` state.json write to include a `pm` context block (feature ID, milestone, progress) in the same atomic write.
5. **Roadmap duplication with GSD** — PM roadmap duplicates GSD ROADMAP.md. Prevent by defining distinct roles: PM roadmap = intent (what to build), GSD ROADMAP.md = execution log (what was built).

## Implications for Roadmap

Based on research, three phases with clear dependency ordering:

### Phase 1: Core PM Primitives
**Rationale:** roadmap.json is the foundational artifact — every other PM feature reads from or writes to it. Must be designed and locked first. Human task inbox depends on a consistent task marker schema. WIP limit and feature-to-pipeline handoff establish the PM-to-pipeline boundary.
**Delivers:** Usable PM system in terminal. Developer can add features, view roadmap, start features through pipeline, see human tasks, close milestones.
**Addresses:** roadmap.json schema, /seraphim:roadmap, /seraphim:add-feature, /seraphim:start, WIP limit, milestone completion/archival, human task inbox (/seraphim:inbox), blocked feature flagging
**Avoids:** Second system effect (tight scope), ceremony creep (PM as read-path, pipeline never blocked)
**Key design decision:** Extend `/seraphim:pause` state.json to include PM context block in this phase, not later — retrofitting is expensive (Pitfall 4).

### Phase 2: Progress Visibility
**Rationale:** Once core primitives exist and features flow through the pipeline, the next need is cross-project visibility and dashboard integration. All dashboard extensions are read-only and can be built independently once data sources exist.
**Delivers:** Cost-to-date per feature/milestone, cross-project overview command, roadmap panel in dashboard, human tasks panel in dashboard, feature dependency checks.
**Uses:** Existing sync script (add two collection targets), existing multi-project-scanner.js (extend output), existing Neon database (two new tables), existing Next.js dashboard (new panels)
**Implements:** Neon feature_snapshots and human_task_snapshots tables, sync script extensions, dashboard PM panels
**Avoids:** Dashboard scan latency (Pitfall 8) — use event-driven PM state via shared event log rather than scanning all projects on every render

### Phase 3: Intelligence Layer
**Rationale:** Requires accumulated data from Phases 1-2 to be meaningful. Velocity trends need completion timestamps. Cost comparisons need multiple completed features. "What needs attention" needs enough projects with PM state to surface useful signals.
**Delivers:** Velocity trends, cross-project cost trends, attention signals, research task type with RAG handoff, GSD ROADMAP.md import, milestone cost tracking
**Avoids:** Premature optimization — these features are valuable only after the PM system has real usage data

### Phase Ordering Rationale

- roadmap.json must exist before any command can read/write PM state (Phase 1 prerequisite for everything)
- Dashboard panels require Neon tables which require sync script extensions which require file schemas to exist (Phase 1 before Phase 2)
- Intelligence features require accumulated decision data from real pipeline runs through the PM layer (Phase 2 before Phase 3)
- Pause/resume PM integration must ship with Phase 1 to avoid costly retrofit (Pitfall 4 prevention)
- Human task inbox in Phase 1 because it aggregates existing SERAPHIM:HUMAN_TASKS markers — low effort, high value

### Research Flags

Phases likely needing deeper research during planning:
- **Phase 1:** roadmap.json schema design needs careful specification — it is the foundational artifact and changing it later forces cascading updates across all PM commands
- **Phase 2:** Event-driven PM state vs file scanning trade-off for dashboard latency needs validation with real project count

Phases with standard patterns (skip research-phase):
- **Phase 1 commands:** Standard CRUD on JSON files, well-documented in GSD source patterns
- **Phase 2 dashboard:** Extends existing Next.js + Neon pattern from v3.0, no new technology
- **Phase 3:** All features are computations over existing data; standard aggregation patterns

## Confidence Assessment

| Area | Confidence | Notes |
|------|------------|-------|
| Stack | HIGH | Zero new packages; all additions verified against existing installed stack |
| Features | HIGH | Grounded in GSD source analysis, existing Seraphim commands, and PM ecosystem research |
| Architecture | HIGH | v3.0 architecture is shipped and verified; v3.1 extends without modifying |
| Pitfalls | HIGH | Second system effect and ceremony creep are canonical risks; file-based state pitfalls verified against ACID literature |

**Overall confidence:** HIGH

### Gaps to Address

- **roadmap.json schema finalization:** Research identifies the need but does not lock the schema. Must be specified in Phase 1 requirements before any implementation.
- **Event-driven vs scan-driven PM state for dashboard:** PITFALLS.md recommends event-driven (shared event log) but STACK.md assumes extending the existing sync script (scan-driven). Resolve during Phase 2 planning.
- **Phase 9 human gate integration:** Human task inbox depends on Phase 9 gate decisions (D-01 through D-03) writing tasks to a consistent schema. If Phase 9 is not yet shipped, the inbox may need a compatibility shim.
- **Definition of "done" per feature type:** Code features close on Crucible pass, but research and mixed features need configurable done signals. Define in Phase 1 requirements.

## Sources

### Primary (HIGH confidence)
- GSD source (live): workflows/new-project.md, new-milestone.md, execute-phase.md, autonomous.md
- Seraphim v3.0 design spec: docs/specs/2026-04-04-seraphim-v3-design.md
- Seraphim existing commands and pause/resume: direct inspection
- Node.js v22, pg package, Neon Postgres: verified on installed system

### Secondary (MEDIUM confidence)
- AIPIM (JSONL event-log PM pattern): https://github.com/rmarsigli/aipim
- Backlog.md: https://github.com/MrLesk/Backlog.md
- Linear, GitHub Projects, Kanban literature — filtered for solo developer context
- Oracle/Richmond Alake: file-based vs database pitfalls for AI agent memory (Feb 2026)

### Tertiary (LOW confidence)
- Complex.so: PM tool failure modes for small teams — single source

---
*Research completed: 2026-04-08*
*Ready for roadmap: yes*
