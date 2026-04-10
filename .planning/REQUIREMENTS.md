# Requirements — v3.2 Idea-to-Shipped Journey

## Foundations

- [x] **FOUND-01**: v3.1 Neon DDL applied — all PM tables exist in production Neon
- [x] **FOUND-02**: Schema consistency — `project` vs `project_name` mismatch resolved across all tables
- [x] **FOUND-03**: `feature_id` flows through decisions-logger to Neon
- [ ] **FOUND-04**: Schema extension audit — every v3.2 data concept extends existing structures (no parallel files)

## Idea Capture

- [ ] **SEED-01**: User can capture a raw idea via `/seraphim:seed` with braindump-style freeform input
- [ ] **SEED-02**: Seeds stored in `.planning/seeds/` with SEED-NNN.md format and index.jsonl for lookups
- [ ] **SEED-03**: User can promote a seed to a feature with requirements via `/seraphim:promote`
- [ ] **SEED-04**: Seeds have trigger conditions that auto-surface during new-milestone when scope matches
- [ ] **SEED-05**: User can capture zero-friction notes via `/seraphim:note` (one write, no questions)
- [ ] **SEED-06**: User can add structured todos via `/seraphim:add-todo` with area tagging
- [ ] **SEED-07**: User can list and select pending todos via `/seraphim:check-todos`

## Requirements

- [ ] **REQ-01**: User can define requirements with REQ-IDs via `/seraphim:requirements` (AI suggests, human approves)
- [ ] **REQ-02**: Requirements grouped by category with v1/future/out-of-scope scoping
- [ ] **REQ-03**: REQ traceability matrix mapping REQ-IDs to phases, features, and verification status
- [ ] **REQ-04**: `lib/requirements.js` manages REQ-ID CRUD following roadmap.js atomic write pattern

## Planning

- [ ] **PLAN-01**: Roadmap.json extended with waves, dependency graph, and success criteria per feature
- [ ] **PLAN-02**: Dependency resolution via Kahn's algorithm in `lib/wave-planner.js`
- [ ] **PLAN-03**: User can generate wave-structured PLAN.md via `/seraphim:plan` with tasks and done-criteria
- [ ] **PLAN-04**: User can lock implementation decisions before planning via `/seraphim:discuss` producing CONTEXT.md
- [ ] **PLAN-05**: User can surface Claude's assumptions about a phase via `/seraphim:assumptions`
- [ ] **PLAN-06**: Plan verification loop — planner + checker agents with revision (max 3 iterations)

## Execution

- [ ] **EXEC-01**: User can execute all plans in a phase via `/seraphim:execute` with wave-based parallel execution
- [ ] **EXEC-02**: User can execute a single plan via `/seraphim:execute-plan`
- [ ] **EXEC-03**: User can run all remaining phases autonomously via `/seraphim:autonomous` (discuss→plan→execute per phase)
- [ ] **EXEC-04**: User can execute small ad-hoc tasks via `/seraphim:quick` with atomic commits and state tracking
- [ ] **EXEC-05**: User can execute trivial tasks inline via `/seraphim:fast` (no subagents, no ceremony)
- [ ] **EXEC-06**: Wave-based parallel execution with dependency analysis and agent grouping

## Research

- [ ] **RSRCH-01**: User can scope research focus via `/seraphim:research-scope` (human interrogation gate)
- [ ] **RSRCH-02**: User can run AI research via `/seraphim:research-run` (only after scope is locked)
- [ ] **RSRCH-03**: Two-command separation enforced — interrogation gate cannot be skipped
- [ ] **RSRCH-04**: `lib/research-tracker.js` manages research item state and categorization
- [ ] **RSRCH-05**: User can analyze codebase structure via `/seraphim:map-codebase` with parallel mapper agents

## Verification

- [ ] **VFY-01**: User can verify built features via `/seraphim:verify` with goal-backward traceability
- [ ] **VFY-02**: Every verification report contains at least one REQUIRES_HUMAN_JUDGMENT item
- [ ] **VFY-03**: User can validate phase completion via `/seraphim:validate` with Nyquist gap auditing
- [ ] **VFY-04**: User can run conversational UAT via `/seraphim:uat` with persistent UAT.md state
- [ ] **VFY-05**: User can audit milestone completion via `/seraphim:audit-milestone` checking cross-phase integration
- [ ] **VFY-06**: User can run cross-phase UAT audit via `/seraphim:audit-uat` surfacing unresolved items

## Debugging

- [ ] **DBG-01**: User can debug systematically via `/seraphim:debug` with persistent state across resets
- [ ] **DBG-02**: Autonomous root-cause analysis agents for UAT gaps
- [ ] **DBG-03**: User can run post-mortem investigation via `/seraphim:forensics` (read-only, diagnostic)
- [ ] **DBG-04**: Failed task auto-repair with RETRY/DECOMPOSE/PRUNE/ESCALATE strategies

## Human Tasks

- [ ] **HTASK-01**: Human task inbox enriched with skills-to-learn field
- [ ] **HTASK-02**: Human task inbox enriched with thought-prompt field for high-leverage thinking
- [ ] **HTASK-03**: Human task inbox enriched with research-task field

## Navigation & Routing

- [ ] **NAV-01**: User can auto-advance to next logical step via `/seraphim:next` (discuss→plan→execute→verify progression)
- [ ] **NAV-02**: User can route freeform text to the right command via `/seraphim:do`
- [ ] **NAV-03**: User can check project progress and route to next action via `/seraphim:progress`

## Session Management

- [ ] **SESS-01**: User can pause work with full context handoff via `/seraphim:pause` (HANDOFF.json + .continue-here.md)
- [ ] **SESS-02**: User can resume work from previous session via `/seraphim:resume` with context restoration
- [ ] **SESS-03**: Session reports generated via `/seraphim:session-report` with work summary and outcomes

## Phase & Milestone Management

- [ ] **MGMT-01**: User can add a phase to end of milestone via `/seraphim:add-phase`
- [ ] **MGMT-02**: User can insert urgent decimal phase between existing phases via `/seraphim:insert-phase`
- [ ] **MGMT-03**: User can remove an unstarted phase via `/seraphim:remove-phase` with renumbering
- [ ] **MGMT-04**: User can complete milestone via `/seraphim:complete-milestone` with archival and git tagging
- [ ] **MGMT-05**: User can create clean PR branch filtering .planning/ via `/seraphim:pr-branch`
- [ ] **MGMT-06**: User can validate .planning/ directory integrity via `/seraphim:health`
- [ ] **MGMT-07**: User can manage parallel workstreams via `/seraphim:workstreams`
- [ ] **MGMT-08**: User can manage phases from interactive command center via `/seraphim:manager`

## Visualization & Reporting

- [ ] **VIZ-01**: Dashboard shows progress bars and completion % per phase and milestone
- [ ] **VIZ-02**: Dashboard shows velocity tracking (rolling 7-day completion rate)
- [ ] **VIZ-03**: Dashboard shows wave progress panels (per-wave breakdown)
- [ ] **VIZ-04**: User can view comprehensive project statistics via `/seraphim:stats`
- [ ] **VIZ-05**: Full roadmap tree view in dashboard with phases/waves/tasks/costs

## Configuration

- [ ] **CFG-01**: Model profiles (quality/balanced/budget/inherit) control agent routing per command
- [ ] **CFG-02**: User can configure workflow settings via `/seraphim:settings`

## UI & Quality

- [ ] **UI-01**: User can generate UI design contract via `/seraphim:ui-spec` for frontend phases
- [ ] **UI-02**: User can run retroactive 6-pillar UI audit via `/seraphim:ui-review`
- [ ] **UI-03**: User can generate tests for completed phase via `/seraphim:add-tests`

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
| FOUND-04 | Phase 32 | Pending |
| SEED-01 | Phase 33 | Pending |
| SEED-02 | Phase 33 | Pending |
| SEED-03 | Phase 33 | Pending |
| SEED-04 | Phase 33 | Pending |
| SEED-05 | Phase 33 | Pending |
| SEED-06 | Phase 33 | Pending |
| SEED-07 | Phase 33 | Pending |
| REQ-01 | Phase 33 | Pending |
| REQ-02 | Phase 33 | Pending |
| REQ-03 | Phase 33 | Pending |
| REQ-04 | Phase 33 | Pending |
| PLAN-01 | Phase 33 | Pending |
| PLAN-02 | Phase 33 | Pending |
| PLAN-03 | Phase 33 | Pending |
| PLAN-04 | Phase 33 | Pending |
| PLAN-05 | Phase 33 | Pending |
| PLAN-06 | Phase 33 | Pending |
| EXEC-01 | Phase 33 | Pending |
| EXEC-02 | Phase 33 | Pending |
| EXEC-03 | Phase 33 | Pending |
| EXEC-04 | Phase 33 | Pending |
| EXEC-05 | Phase 33 | Pending |
| EXEC-06 | Phase 33 | Pending |
| RSRCH-01 | Phase 34 | Pending |
| RSRCH-02 | Phase 34 | Pending |
| RSRCH-03 | Phase 34 | Pending |
| RSRCH-04 | Phase 34 | Pending |
| RSRCH-05 | Phase 34 | Pending |
| SESS-01 | Phase 34 | Pending |
| SESS-02 | Phase 34 | Pending |
| SESS-03 | Phase 34 | Pending |
| NAV-01 | Phase 34 | Pending |
| NAV-02 | Phase 34 | Pending |
| NAV-03 | Phase 34 | Pending |
| MGMT-01 | Phase 35 | Pending |
| MGMT-02 | Phase 35 | Pending |
| MGMT-03 | Phase 35 | Pending |
| MGMT-04 | Phase 35 | Pending |
| MGMT-05 | Phase 35 | Pending |
| MGMT-06 | Phase 35 | Pending |
| MGMT-07 | Phase 35 | Pending |
| MGMT-08 | Phase 35 | Pending |
| CFG-01 | Phase 35 | Pending |
| CFG-02 | Phase 35 | Pending |
| UI-01 | Phase 35 | Pending |
| UI-02 | Phase 35 | Pending |
| UI-03 | Phase 35 | Pending |
| HTASK-01 | Phase 36 | Pending |
| HTASK-02 | Phase 36 | Pending |
| HTASK-03 | Phase 36 | Pending |
| DBG-01 | Phase 36 | Pending |
| DBG-02 | Phase 36 | Pending |
| DBG-03 | Phase 36 | Pending |
| DBG-04 | Phase 36 | Pending |
| VFY-01 | Phase 37 | Pending |
| VFY-02 | Phase 37 | Pending |
| VFY-03 | Phase 37 | Pending |
| VFY-04 | Phase 37 | Pending |
| VFY-05 | Phase 37 | Pending |
| VFY-06 | Phase 37 | Pending |
| VIZ-01 | Phase 37 | Pending |
| VIZ-02 | Phase 37 | Pending |
| VIZ-03 | Phase 37 | Pending |
| VIZ-04 | Phase 37 | Pending |
| VIZ-05 | Phase 37 | Pending |

---
*v3.2 — 69 requirements across 15 categories*
*Generated: 2026-04-09*
