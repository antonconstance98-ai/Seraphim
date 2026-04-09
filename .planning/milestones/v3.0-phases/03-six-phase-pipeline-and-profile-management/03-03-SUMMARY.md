---
phase: 03-six-phase-pipeline-and-profile-management
plan: "03"
subsystem: pipeline
tags: [seraphim, judge, architect, pipeline, verdicts, blueprint, markers, dispatch]

requires:
  - phase: 03-01-SUMMARY.md
    provides: markers.js, banner.js, config.js, dispatch.js, phase-state.js

provides:
  - judge.md slash command — Wing III adversarial stress-test with SURVIVES/FATAL_FLAW/CONDITIONAL verdicts
  - architect.md slash command — Wing IV blueprint creation with BLUEPRINT and TASK markers
  - Loop cap enforcement in judge (judge_envision counter vs max_loops)
  - project_type propagation from config into BLUEPRINT marker (per D-03)

affects: [forge, crucible, pipeline-execution, phase-routing]

tech-stack:
  added: []
  patterns:
    - "profile_at_execution recorded to state.json BEFORE model dispatch (per D-05)"
    - "Conditional inline dispatch: claude-opus-4-6 executes directly, all other models use CLI subprocess"
    - "All phase outputs validated with parseMarkers() before accept — unparseable files abort with error"
    - "loop_required flag in PHASE_COMPLETE signals Judge-Envision feedback loop to downstream phases"
    - "project_type flows from config.json into BLUEPRINT marker, enabling Forge per-task branching"

key-files:
  created:
    - ~/.claude/plugins/seraphim/commands/judge.md
    - ~/.claude/plugins/seraphim/commands/architect.md
  modified: []

key-decisions:
  - "loop_required=true only when ALL approaches receive FATAL_FLAW (survivors=0 and conditional=0) — not when some conditionals exist"
  - "Architect selects SURVIVES first, CONDITIONAL fallback — CONDITIONAL selection is noted in blueprint overview"
  - "All TASK markers include type attribute (code/prose/analysis) even for homogeneous projects — enables Forge per-task branching for mixed types"
  - "Blueprint validation requires both BLUEPRINT marker and at least one TASK marker — missing either aborts"

patterns-established:
  - "Phase commands follow: resolve root -> check prerequisite -> read config -> audit state -> check caps -> dispatch -> validate output -> print banner"
  - "prerequisite chain: vision.md (envision) -> judgment.md (judge) -> blueprint.md (architect)"

requirements-completed: [PIPE-03, PIPE-04, PIPE-10]

duration: 8min
completed: 2026-04-08
---

# Phase 03 Plan 03: Judge and Architect Commands Summary

**Wing III (Judge) adversarial verdict system with SURVIVES/FATAL_FLAW/CONDITIONAL markers and Wing IV (Architect) blueprint generator with project_type-aware TASK breakdown**

## Performance

- **Duration:** 8 min
- **Started:** 2026-04-08T21:40:00Z
- **Completed:** 2026-04-08T21:48:00Z
- **Tasks:** 2
- **Files modified:** 2

## Accomplishments

- Judge command checks vision.md prerequisite, enforces judge_envision loop cap, resolves model via dispatch, dispatches inline (Opus) or via CLI, validates output markers, prints Wing III banner with verdict counts
- Architect command checks judgment.md for survivors, selects top approach, reads project_type from config, dispatches inline or via CLI, produces blueprint with BLUEPRINT + TASK markers, validates output, prints Wing IV banner
- Both commands record profile audit to state.json before execution per D-05
- Prerequisite chain enforced: judge requires vision.md, architect requires judgment.md with at least one survivor

## Task Commits

1. **Task 1: Judge command** - `80b0a73` (feat) — plugin repo
2. **Task 2: Architect command with project_type support** - `40177ba` (feat) — plugin repo

## Files Created/Modified

- `/home/alucardmessangeroflight/.claude/plugins/seraphim/commands/judge.md` — Wing III adversarial judge command
- `/home/alucardmessangeroflight/.claude/plugins/seraphim/commands/architect.md` — Wing IV blueprint architect command

## Decisions Made

- `loop_required=true` only when ALL approaches receive FATAL_FLAW — conditionals count as viable paths
- Architect CONDITIONAL fallback noted in blueprint overview section to surface the condition to downstream phases
- TASK markers include `type` attribute on every task to allow Forge to branch per-task for mixed project types

## Deviations from Plan

None - plan executed exactly as written.

## Issues Encountered

None.

## User Setup Required

None - no external service configuration required.

## Next Phase Readiness

- Wings III and IV complete — judge.md and architect.md available as slash commands
- Prerequisite chain intact: envision -> judge -> architect -> forge
- project_type flows from config into BLUEPRINT marker, ready for Forge branching in plan 04

---
*Phase: 03-six-phase-pipeline-and-profile-management*
*Completed: 2026-04-08*

## Self-Check: PASSED

- `/home/alucardmessangeroflight/.claude/plugins/seraphim/commands/judge.md` — FOUND
- `/home/alucardmessangeroflight/.claude/plugins/seraphim/commands/architect.md` — FOUND
- Plugin repo commit `80b0a73` — FOUND (judge.md)
- Plugin repo commit `40177ba` — FOUND (architect.md)
