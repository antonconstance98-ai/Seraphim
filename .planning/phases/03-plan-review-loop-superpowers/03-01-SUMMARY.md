---
phase: 03-plan-review-loop-superpowers
plan: 01
subsystem: hooks
tags: [codex, multi-round-review, plan-review, hooks, node.js, adversarial-review]

# Dependency graph
requires:
  - phase: 02-review-gate-gsd-integration
    provides: "codex-exec.js wrapper, single-pass codex-plan-reviewer.js, SubagentStop hook registration"
provides:
  - "codex-multi-round-reviewer.js — shared multi-round loop orchestrator with constructive + adversarial rounds"
  - "Upgraded codex-plan-reviewer.js v3.0.0 calling multi-round reviewer instead of single-pass"
  - "review-state.json persistence schema for interrupted session resumption"
  - "Typed HANDOFF.md spec with Decisions Not Taken section"
  - "SubagentStop hook timeout increased from 300s to 600s"
affects:
  - 03-plan-review-loop-superpowers (plan 02 — Superpowers integration reuses codex-multi-round-reviewer.js)
  - Any GSD planning subagent that triggers SubagentStop

# Tech tracking
tech-stack:
  added: []
  patterns:
    - "Multi-round Codex review: constructive round first, adversarial round only if issues found"
    - "Early exit optimization: skip adversarial round when round 1 finds zero issues"
    - "Review state persistence: write state BEFORE each Codex call to enable session resumption"
    - "Fail-open error handling: SubagentStop hook always exits 0 on Codex failures"
    - "Typed handoff spec: HANDOFF.md with Decisions Not Taken section preserves Opus authority"

key-files:
  created:
    - "~/.claude/hooks/codex-multi-round-reviewer.js — 575-line multi-round loop orchestrator module"
  modified:
    - "~/.claude/hooks/codex-plan-reviewer.js — upgraded from v2.0.0 to v3.0.0, now calls multi-round reviewer"
    - "~/.claude/settings.json — SubagentStop timeout increased from 300 to 600"

key-decisions:
  - "Two distinct Codex prompts: constructive (find issues) vs adversarial (poke holes) — prevents Round 2 being redundant"
  - "Round 2 only fires if Round 1 found issues — early exit saves tokens on clean plans (D-02)"
  - "State written BEFORE each Codex call to prevent re-entry on crash/restart (Pitfall 1 mitigation)"
  - "reviewsPath returned null from multi-round reviewer — caller (codex-plan-reviewer.js) writes REVIEWS.md to maintain single responsibility"
  - "Decisions Not Taken section pre-populated with placeholder — caller populates after Opus final revision (per D-03 Opus authority)"

patterns-established:
  - "Pattern: shared reviewer module — codex-multi-round-reviewer.js is a standalone module requiring codex-exec.js, reusable by Superpowers"
  - "Pattern: state-before-call — write review-state.json before invoking Codex to survive interruptions"
  - "Pattern: stderr for progress, stdout for hook JSON — milestone updates go to stderr only"
  - "Pattern: fail-open — SubagentStop always exits 0 on error, block only on HIGH severity"

requirements-completed: [REVW-03, REVW-04, REVW-06]

# Metrics
duration: 4min
completed: 2026-04-02
---

# Phase 03 Plan 01: Multi-Round Plan Review Loop Summary

**2-round Opus-Codex plan review loop with constructive + adversarial Codex passes, early exit on clean plans, review-state.json persistence, and typed HANDOFF.md spec with Decisions Not Taken section**

## Performance

- **Duration:** 4 min
- **Started:** 2026-04-02T20:51:33Z
- **Completed:** 2026-04-02T20:55:42Z
- **Tasks:** 2
- **Files modified:** 3 (2 hook files + settings.json)

## Accomplishments

- Created `codex-multi-round-reviewer.js` (575 lines) — a shared module implementing the 5-step multi-round review loop with constructive and adversarial Codex passes, review-state.json persistence, HANDOFF.md writer, token logging per round, and early exit when no issues found
- Upgraded `codex-plan-reviewer.js` from v2.0.0 to v3.0.0 — now calls `runMultiRoundReview` from the new module instead of a single-pass Codex call; all existing helper functions preserved; standalone mode updated to show round count, early exit status, and handoff spec path
- Updated `~/.claude/settings.json` SubagentStop timeout from 300s to 600s to accommodate 2 rounds at 180s each plus startup overhead

## Task Commits

Tasks committed to filesystem (hook files at ~/.claude/hooks/ are not tracked in project git repo — same pattern as Phases 1 and 2):

1. **Task 1: Create codex-multi-round-reviewer.js** — created at `~/.claude/hooks/codex-multi-round-reviewer.js`
2. **Task 2: Upgrade codex-plan-reviewer.js + update settings.json** — modified `~/.claude/hooks/codex-plan-reviewer.js` and `~/.claude/settings.json`

**Plan metadata commit:** Recorded in project git repo (SUMMARY.md, STATE.md, ROADMAP.md)

## Files Created/Modified

- `~/.claude/hooks/codex-multi-round-reviewer.js` — New shared multi-round loop orchestrator (575 lines). Exports `runMultiRoundReview`. Implements constructive + adversarial review rounds with state persistence, token logging, handoff spec writing, and early exit optimization
- `~/.claude/hooks/codex-plan-reviewer.js` — Upgraded to v3.0.0. Now delegates to `runMultiRoundReview` instead of single-pass review. Returns multi-round aware additionalContext/block responses. All 5 helper functions preserved. Standalone mode shows round count + handoff path
- `~/.claude/settings.json` — SubagentStop hook timeout changed from 300 to 600. All other hooks and settings unchanged

## Decisions Made

- **Two distinct prompts (CONSTRUCTIVE_PROMPT vs ADVERSARIAL_PROMPT):** The constructive prompt asks Codex to find actionable issues with `[SEVERITY: HIGH|MEDIUM|LOW]` tags; the adversarial prompt asks Codex to poke holes and find what can go wrong with `[CONCERN]` tags. Distinct tag formats prevent the adversarial round from being confused with constructive findings when parsing
- **reviewsPath is null from runMultiRoundReview:** The multi-round module doesn't write REVIEWS.md — that's done by `codex-plan-reviewer.js` which calls `writeReviewsFile`. This maintains single responsibility and lets the Superpowers reviewer (plan 02) do the same
- **State write before Codex call:** `advanceRound()` writes state to disk before spawning the Codex subprocess. If the hook is killed mid-review, the next invocation resumes from the correct round rather than restarting

## Deviations from Plan

None — plan executed exactly as written.

## Issues Encountered

None — all acceptance criteria passed on first implementation.

## User Setup Required

None — no external service configuration required. All changes are to existing hook infrastructure.

## Next Phase Readiness

- `codex-multi-round-reviewer.js` is ready to be reused by plan 02 (Superpowers integration) via `require('./codex-multi-round-reviewer')`
- SubagentStop hook is now wired to the multi-round loop and will trigger on any gsd-planner subagent completing
- `review-state.json` will be created at `.planning/review-state.json` in the project root on first real review run

---
*Phase: 03-plan-review-loop-superpowers*
*Completed: 2026-04-02*
