---
phase: 03-six-phase-pipeline-and-profile-management
plan: 02
subsystem: pipeline
tags: [seraphim, discover, envision, claude-commands, agents, SERAPHIM-markers, wing-banners]

requires:
  - phase: 03-01
    provides: dispatch.js, markers.js, banner.js, phase-state.js, config.js — all consumed directly

provides:
  - /seraphim:discover command — runs external + internal research tracks, writes discovery/ output
  - /seraphim:envision command — reads discovery, generates 3-5 approaches in vision.md
  - seraphim-discover agent — Opus role definition for inline discover execution

affects: [03-03-judge, 03-04-architect, 03-05-forge, 03-06-crucible, pipeline-integration]

tech-stack:
  added: []
  patterns:
    - "Inline-vs-dispatch split: commands check resolveExecutorId; if claude-opus-4-6 run inline, else CLI dispatch"
    - "Profile audit before execution: write profile_at_execution + model_at_execution to state.json before any track runs"
    - "TRACK_FAILED stubs: on dispatch failure write stub file with SERAPHIM:TRACK_FAILED marker so downstream phases can still proceed"
    - "Prerequisite chain: each command checks for prior phase output files before executing"

key-files:
  created:
    - ~/.claude/plugins/seraphim/commands/discover.md
    - ~/.claude/plugins/seraphim/commands/envision.md
    - ~/.claude/plugins/seraphim/agents/seraphim-discover.md
  modified: []

key-decisions:
  - "discover.md runs tracks sequentially (external first, internal second) — true parallelism deferred to future enhancement per Open Question 1"
  - "envision.md aborts on missing discovery files (no stubs) — missing discovery is a hard prerequisite failure, unlike track failures within discovery"
  - "seraphim-discover.md agent documents SERAPHIM marker format inline — agent has full context without loading external files"

patterns-established:
  - "Command files instruct Opus step-by-step using Bash node -e snippets for all lib calls"
  - "Wing banner printed at command end via renderWingBanner with model/profile/status/cost/phaseId"
  - "Phase-id argument always validated first; project root resolved second before any file ops"

requirements-completed: [PIPE-01, PIPE-02]

duration: 15min
completed: 2026-04-08
---

# Phase 03 Plan 02: Discover and Envision Commands Summary

**Wing I (Discover) and Wing II (Envision) slash commands with inline/dispatch split, TRACK_FAILED stub resilience, and profile audit trail**

## Performance

- **Duration:** ~15 min
- **Started:** 2026-04-08T21:35:00Z
- **Completed:** 2026-04-08T21:50:00Z
- **Tasks:** 2
- **Files modified:** 3

## Accomplishments

- discover.md: 9-step command that resolves models, runs external + internal research tracks sequentially, writes discovery/ output files with SERAPHIM markers, handles dispatch failures with TRACK_FAILED stubs, audits profile before execution, prints Wing I banner
- envision.md: 9-step command that checks prerequisite discovery files, resolves envision model, generates vision.md inline when Opus is assigned (avoiding Pitfall 2 — Opus-calling-Opus via CLI), dispatches to external models otherwise, writes 3-5 APPROACH markers, prints Wing II banner
- seraphim-discover.md: agent definition giving Opus its Discoverer role, marker format spec, and structured output templates for external.md and internal.md

## Task Commits

1. **Task 1: Discover command and agent** — `7b09959` (feat) — plugin repo
2. **Task 2: Envision command** — `b2a8b17` (feat) — plugin repo

## Files Created/Modified

- `~/.claude/plugins/seraphim/commands/discover.md` — /seraphim:discover command (Wing I)
- `~/.claude/plugins/seraphim/commands/envision.md` — /seraphim:envision command (Wing II)
- `~/.claude/plugins/seraphim/agents/seraphim-discover.md` — Opus discoverer agent definition

## Decisions Made

- Sequential track execution in discover (external first, internal second) — per Open Question 1 from research, true parallelism deferred
- Envision aborts on missing discovery files (unlike discover which stubs failed tracks) — missing input is an unrecoverable prerequisite failure
- Agent file documents SERAPHIM marker format inline so the agent has full context at execution time

## Deviations from Plan

None — plan executed exactly as written.

## Issues Encountered

None.

## User Setup Required

None — no external service configuration required.

## Next Phase Readiness

- Wing I and Wing II commands are ready; pipeline can run discover → envision
- Next: 03-03 Judge command (Wing III) reads vision.md and evaluates approaches
- Prerequisite chain for Judge: envision must have written vision.md with APPROACH markers

---
*Phase: 03-six-phase-pipeline-and-profile-management*
*Completed: 2026-04-08*
