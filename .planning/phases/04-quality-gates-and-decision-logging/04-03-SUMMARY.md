---
phase: 04-quality-gates-and-decision-logging
plan: "03"
subsystem: command-layer
tags: [judge-envision-loop, decisions-logging, quality-gates, loop-escalation]
dependency_graph:
  requires: [04-01]
  provides: [judge-envision-loop-wiring, decisions-jsonl-write-on-judge]
  affects: [envision.md, judge.md]
tech_stack:
  added: []
  patterns: [parseMarkers-on-judgment, appendDecision-after-phase-complete, startMs-latency-capture]
key_files:
  created:
    - ~/.claude/plugins/seraphim/lib/decisions-logger.js
  modified:
    - ~/.claude/plugins/seraphim/commands/envision.md
    - ~/.claude/plugins/seraphim/commands/judge.md
decisions:
  - "[Phase 04-03]: decisions-logger.js created here (was planned in 04-01 but missing) — Rule 3 auto-fix"
  - "[Phase 04-03]: loopContext injected via bash parameter expansion in dispatch path: ${LOOP_CONTEXT:+...}"
  - "[Phase 04-03]: judgeKillRate = fatal / approaches.length — null when approaches array is empty"
metrics:
  duration_min: 4
  completed_date: "2026-04-08"
  tasks_completed: 2
  files_changed: 3
---

# Phase 04 Plan 03: Judge-Envision Loop Wiring and Decisions Logging Summary

**One-liner:** Judge->Envision feedback loop wired via judgment.md marker detection with decisions.jsonl write on every judge completion including judge_kill_rate quality signal.

## Tasks Completed

| Task | Name | Commit | Files |
|------|------|--------|-------|
| 1 | Add loop detection and findings injection to envision.md | 4b4deac | commands/envision.md, lib/decisions-logger.js |
| 2 | Add decisions.jsonl logging and loop-cap escalation to judge.md | c0b7432 | commands/judge.md |

## What Was Built

### Task 1: envision.md loop detection (QUAL-03)

Added **Step 3b** between Step 3 and Step 4 in envision.md. The step:

1. Checks if `judgment.md` exists for the current phase
2. Parses it using `markers.js` for a `PHASE_COMPLETE` marker with `loop_required='true'`
3. If found, collects all `FATAL_FLAW` approach markers and builds a `loopContext` block with `## Previous Judge Findings` header, listing each fatal approach with its reason
4. Warns the user if this is the last allowed loop (`loopCount >= maxLoops - 1`)

The `loopContext` is then injected in Step 6:
- **Inline Opus path:** prepended after `PHASE_START` marker with a `SERAPHIM:LOOP_CONTEXT` marker
- **Dispatch path:** injected at the top of the prompt heredoc via `${LOOP_CONTEXT:+...}` bash parameter expansion

### Task 2: judge.md decisions logging and escalation (COST-03, COST-04, QUAL-05)

Two changes to judge.md:

**Step 5 escalation (QUAL-05):** Replaced the plain halt message with a richer block that reads `judgment.md`, extracts fatal findings, and presents three concrete resolution options to the user: revise vision.md, re-run discover+envision, or manually reset the loop counter in state.json.

**Step 9b decisions write (COST-03, COST-04):** After judgment.md is validated and the loop counter is incremented, calls `appendDecision` with:
- Full timing via `startMs = Date.now()` recorded at Step 3 start
- `judge_kill_rate = fatal / approaches.length` quality signal
- `loop_trigger_reason = 'all_approaches_fatal'` when loop is required
- `outcome = 'loop_required' | 'success'`

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 3 - Blocking] Created missing decisions-logger.js**
- **Found during:** Pre-execution check before Task 2
- **Issue:** `decisions-logger.js` was planned in 04-01 but not present in `lib/` — Task 2 requires `appendDecision` and `buildRecord` from it
- **Fix:** Created `~/.claude/plugins/seraphim/lib/decisions-logger.js` using the exact implementation specified in 04-01-PLAN.md Task 2 action block
- **Files modified:** `~/.claude/plugins/seraphim/lib/decisions-logger.js` (new)
- **Commit:** 4b4deac (bundled with Task 1)

## Known Stubs

None — both commands now reference live library modules with no placeholder values flowing to output.

## Self-Check: PASSED

Files exist:
- `~/.claude/plugins/seraphim/commands/envision.md` — FOUND
- `~/.claude/plugins/seraphim/commands/judge.md` — FOUND
- `~/.claude/plugins/seraphim/lib/decisions-logger.js` — FOUND

Commits exist:
- `4b4deac` — FOUND
- `c0b7432` — FOUND
