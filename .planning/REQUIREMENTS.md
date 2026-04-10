# Requirements — v3.2 Idea-to-Shipped Journey

## Foundations

- [x] **FOUND-01**: v3.1 Neon DDL applied — all PM tables exist in production Neon
- [x] **FOUND-02**: Schema consistency — `project` vs `project_name` mismatch resolved across all tables
- [x] **FOUND-03**: `feature_id` flows through decisions-logger to Neon
- [x] **FOUND-04**: Schema extension audit — every v3.2 data concept extends existing structures (no parallel files)

## Idea Capture

- [x] **SEED-01**: User can capture a raw idea via `/seraphim:seed` with braindump-style freeform input
- [x] **SEED-02**: Seeds stored in `.planning/seeds/` with SEED-NNN.md format and index.jsonl for lookups
- [x] **SEED-03**: User can promote a seed to a feature with requirements via `/seraphim:promote`
- [x] **SEED-04**: Seeds have trigger conditions that auto-surface during new-milestone when scope matches
- [x] **SEED-05**: User can capture zero-friction notes via `/seraphim:note` (one write, no questions)
- [x] **SEED-06**: User can add structured todos via `/seraphim:add-todo` with area tagging
- [x] **SEED-07**: User can list and select pending todos via `/seraphim:check-todos`

## Requirements

- [x] **REQ-01**: User can define requirements with REQ-IDs via `/seraphim:requirements` (AI suggests, human approves)
- [x] **REQ-02**: Requirements grouped by category with v1/future/out-of-scope scoping
- [x] **REQ-03**: REQ traceability matrix mapping REQ-IDs to phases, features, and verification status
- [x] **REQ-04**: `lib/requirements.js` manages REQ-ID CRUD following roadmap.js atomic write pattern

## Planning

- [x] **PLAN-01**: Roadmap.json extended with waves, dependency graph, and success criteria per feature
- [x] **PLAN-02**: Dependency resolution via Kahn's algorithm in `lib/wave-planner.js`
- [x] **PLAN-03**: User can generate wave-structured PLAN.md via `/seraphim:plan` with tasks and done-criteria
- [x] **PLAN-04**: User can lock implementation decisions before planning via `/seraphim:discuss` producing CONTEXT.md
- [x] **PLAN-05**: User can surface Claude's assumptions about a phase via `/seraphim:assumptions`
- [x] **PLAN-06**: Plan verification loop — planner + checker agents with revision (max 3 iterations)

## Execution

- [x] **EXEC-01**: User can execute all plans in a phase via `/seraphim:execute` with wave-based parallel execution
- [x] **EXEC-02**: User can execute a single plan via `/seraphim:execute-plan`
- [x] **EXEC-03**: User can run all remaining phases autonomously via `/seraphim:autonomous` (discuss→plan→execute per phase)
- [x] **EXEC-04**: User can execute small ad-hoc tasks via `/seraphim:quick` with atomic commits and state tracking
- [x] **EXEC-05**: User can execute trivial tasks inline via `/seraphim:fast` (no subagents, no ceremony)
- [x] **EXEC-06**: Wave-based parallel execution with dependency analysis and agent grouping

## Research

- [x] **RSRCH-01**: User can scope research focus via `/seraphim:research-scope` (human interrogation gate)
- [x] **RSRCH-02**: User can run AI research via `/seraphim:research-run` (only after scope is locked)
- [x] **RSRCH-03**: Two-command separation enforced — interrogation gate cannot be skipped
- [x] **RSRCH-04**: `lib/research-tracker.js` manages research item state and categorization
- [x] **RSRCH-05**: User can analyze codebase structure via `/seraphim:map-codebase` with parallel mapper agents

## Verification

- [x] **VFY-01**: User can verify built features via `/seraphim:verify` with goal-backward traceability
- [x] **VFY-02**: Every verification report contains at least one REQUIRES_HUMAN_JUDGMENT item
- [x] **VFY-03**: User can validate phase completion via `/seraphim:validate` with Nyquist gap auditing
- [x] **VFY-04**: User can run conversational UAT via `/seraphim:uat` with persistent UAT.md state
- [x] **VFY-05**: User can audit milestone completion via `/seraphim:audit-milestone` checking cross-phase integration
- [x] **VFY-06**: User can run cross-phase UAT audit via `/seraphim:audit-uat` surfacing unresolved items

## Debugging

- [x] **DBG-01**: User can debug systematically via `/seraphim:debug` with persistent state across resets
- [x] **DBG-02**: Autonomous root-cause analysis agents for UAT gaps
- [x] **DBG-03**: User can run post-mortem investigation via `/seraphim:forensics` (read-only, diagnostic)
- [x] **DBG-04**: Failed task auto-repair with RETRY/DECOMPOSE/PRUNE/ESCALATE strategies

## Human Tasks

- [x] **HTASK-01**: Human task inbox enriched with skills-to-learn field
- [x] **HTASK-02**: Human task inbox enriched with thought-prompt field for high-leverage thinking
- [x] **HTASK-03**: Human task inbox enriched with research-task field

## Navigation & Routing

- [x] **NAV-01**: User can auto-advance to next logical step via `/seraphim:next` (discuss→plan→execute→verify progression)
- [x] **NAV-02**: User can route freeform text to the right command via `/seraphim:do`
- [x] **NAV-03**: User can check project progress and route to next action via `/seraphim:progress`

## Session Management

- [x] **SESS-01**: User can pause work with full context handoff via `/seraphim:pause` (HANDOFF.json + .continue-here.md)
- [x] **SESS-02**: User can resume work from previous session via `/seraphim:resume` with context restoration
- [x] **SESS-03**: Session reports generated via `/seraphim:session-report` with work summary and outcomes

## Phase & Milestone Management

- [x] **MGMT-01**: User can add a phase to end of milestone via `/seraphim:add-phase`
- [x] **MGMT-02**: User can insert urgent decimal phase between existing phases via `/seraphim:insert-phase`
- [x] **MGMT-03**: User can remove an unstarted phase via `/seraphim:remove-phase` with renumbering
- [x] **MGMT-04**: User can complete milestone via `/seraphim:complete-milestone` with archival and git tagging
- [x] **MGMT-05**: User can create clean PR branch filtering .planning/ via `/seraphim:pr-branch`
- [x] **MGMT-06**: User can validate .planning/ directory integrity via `/seraphim:health`
- [x] **MGMT-07**: User can manage parallel workstreams via `/seraphim:workstreams`
- [x] **MGMT-08**: User can manage phases from interactive command center via `/seraphim:manager`

## Visualization & Reporting

- [x] **VIZ-01**: Dashboard shows progress bars and completion % per phase and milestone
- [ ] **VIZ-02**: Dashboard shows velocity tracking (rolling 7-day completion rate)
- [x] **VIZ-03**: Dashboard shows wave progress panels (per-wave breakdown)
- [x] **VIZ-04**: User can view comprehensive project statistics via `/seraphim:stats`
- [ ] **VIZ-05**: Full roadmap tree view in dashboard with phases/waves/tasks/costs

## Configuration

- [x] **CFG-01**: Model profiles (quality/balanced/budget/inherit) control agent routing per command
- [x] **CFG-02**: User can configure workflow settings via `/seraphim:settings`

## UI & Quality

- [x] **UI-01**: User can generate UI design contract via `/seraphim:ui-spec` for frontend phases
- [x] **UI-02**: User can run retroactive 6-pillar UI audit via `/seraphim:ui-review`
- [x] **UI-03**: User can generate tests for completed phase via `/seraphim:add-tests`

## Future Requirements (deferred to v3.3+)

- Dashboard click-to-action control center (requires local socket listener)
- Real-time streaming between models

## Out of Scope

- Modifying Claude Code itself — plugin commands, agents, and lib only
- Running MiniMax locally (API-only)
- Auto-applying model changes without human approval
- Supporting models outside the nine-model roster without explicit request

## Traceability

| REQ-ID | Phase | Status |
|--------|-------|--------|
| FOUND-01 | Phase 32 | Complete |
| FOUND-02 | Phase 32 | Complete |
| FOUND-03 | Phase 32 | Complete |
| FOUND-04 | Phase 32 | Complete |
| SEED-01 | Phase 33 | Complete |
| SEED-02 | Phase 33 | Complete |
| SEED-03 | Phase 33 | Complete |
| SEED-04 | Phase 33 | Complete |
| SEED-05 | Phase 33 | Complete |
| SEED-06 | Phase 33 | Complete |
| SEED-07 | Phase 33 | Complete |
| REQ-01 | Phase 33 | Complete |
| REQ-02 | Phase 33 | Complete |
| REQ-03 | Phase 33 | Complete |
| REQ-04 | Phase 33 | Complete |
| PLAN-01 | Phase 33 | Complete |
| PLAN-02 | Phase 33 | Complete |
| PLAN-03 | Phase 33 | Complete |
| PLAN-04 | Phase 33 | Complete |
| PLAN-05 | Phase 33 | Complete |
| PLAN-06 | Phase 33 | Complete |
| EXEC-01 | Phase 33 | Complete |
| EXEC-02 | Phase 33 | Complete |
| EXEC-03 | Phase 33 | Complete |
| EXEC-04 | Phase 33 | Complete |
| EXEC-05 | Phase 33 | Complete |
| EXEC-06 | Phase 33 | Complete |
| RSRCH-01 | Phase 34 | Complete |
| RSRCH-02 | Phase 34 | Complete |
| RSRCH-03 | Phase 34 | Complete |
| RSRCH-04 | Phase 34 | Complete |
| RSRCH-05 | Phase 34 | Complete |
| SESS-01 | Phase 34 | Complete |
| SESS-02 | Phase 34 | Complete |
| SESS-03 | Phase 34 | Complete |
| NAV-01 | Phase 34 | Complete |
| NAV-02 | Phase 34 | Complete |
| NAV-03 | Phase 34 | Complete |
| MGMT-01 | Phase 35 | Complete |
| MGMT-02 | Phase 35 | Complete |
| MGMT-03 | Phase 35 | Complete |
| MGMT-04 | Phase 35 | Complete |
| MGMT-05 | Phase 35 | Complete |
| MGMT-06 | Phase 35 | Complete |
| MGMT-07 | Phase 35 | Complete |
| MGMT-08 | Phase 35 | Complete |
| CFG-01 | Phase 35 | Complete |
| CFG-02 | Phase 35 | Complete |
| UI-01 | Phase 35 | Complete |
| UI-02 | Phase 35 | Complete |
| UI-03 | Phase 35 | Complete |
| HTASK-01 | Phase 36 | Complete |
| HTASK-02 | Phase 36 | Complete |
| HTASK-03 | Phase 36 | Complete |
| DBG-01 | Phase 36 | Complete |
| DBG-02 | Phase 36 | Complete |
| DBG-03 | Phase 36 | Complete |
| DBG-04 | Phase 36 | Complete |
| VFY-01 | Phase 37 | Complete |
| VFY-02 | Phase 37 | Complete |
| VFY-03 | Phase 37 | Complete |
| VFY-04 | Phase 37 | Complete |
| VFY-05 | Phase 37 | Complete |
| VFY-06 | Phase 37 | Complete |
| VIZ-01 | Phase 37 | Complete |
| VIZ-02 | Phase 37 | Pending |
| VIZ-03 | Phase 37 | Complete |
| VIZ-04 | Phase 37 | Complete |
| VIZ-05 | Phase 37 | Pending |

---
*v3.2 — 69 requirements across 15 categories*
*Generated: 2026-04-09*
