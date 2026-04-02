---
phase: 02-review-gate-gsd-integration
plan: "01"
subsystem: hooks
tags: [stop-hook, review-gate, codex-review, routing, allow-block-protocol, token-logging]

dependency_graph:
  requires:
    - phase: 01-foundation
      provides: codex-exec.js (runCodexExec, parseCodexTokens, computeCost), codex-token-logger.js (JSONL record schema), codex-router.js (Phase 1 routing hook), hook registration in settings.json
  provides:
    - codex-review-gate.js (Stop hook review gate with ALLOW/BLOCK protocol and task-type-aware review depth)
    - codex-router.js extended to global opt-out routing (ROUT-02)
    - Stop hook registered in ~/.claude/settings.json with 300s timeout
  affects:
    - All future Claude turns that modify code files (Stop hook gates every completion)
    - All future Write/Edit tool calls in any Claude workflow (global routing advisory)
    - 02-02 (GSD wave-boundary hook — builds on same Stop hook infrastructure)

tech-stack:
  added: []
  patterns:
    - Stop hook pattern with stop_hook_active infinite loop guard
    - Code change detection via git diff HEAD + git status --porcelain (untracked files via Pitfall 2 mitigation)
    - Task-type classification: security > test-gen > bulk-ops > feature (D-12)
    - Depth-varied review prompts: deep 4-category for feature/security, light ALLOW/BLOCK-only for test-gen/bulk-ops
    - git diff -U10 for 10 lines of surrounding context (D-11)
    - ALLOW/BLOCK verdict scanning all output lines (Pitfall 4 mitigation)
    - Fail-open pattern: process.exit(0) on any error (never block user on Codex failure)
    - Global opt-out routing: fires for all projects unless routing_disabled === true (Phase 2 ROUT-02)

key-files:
  created:
    - /home/alucard/.claude/hooks/codex-review-gate.js
  modified:
    - /home/alucard/.claude/hooks/codex-router.js
    - /home/alucard/.claude/settings.json

key-decisions:
  - "Fail-open on all errors: process.exit(0) in outer catch — never block user when Codex is unavailable (ROUT-03 compatible)"
  - "Phase 2 routing is opt-out not opt-in: fires for all configured projects unless routing_disabled=true; preserves Phase 1 backward compat for projects with routing_enabled=false"
  - "Stop hook stdin timeout set to 300s (matches hook timeout) — review calls can take 30-120s"

patterns-established:
  - "Stop hook guard pattern: check stop_hook_active FIRST before any other processing (prevents review-triggers-review loops)"
  - "Diff truncation at 8000 chars: leaves room for prompt overhead within 10,000 char additionalContext limit"
  - "First-line verdict scanning: parse ALL output lines for ALLOW/BLOCK (Codex may prefix with preamble)"

requirements-completed: [REVW-01, REVW-02, ROUT-02, GSD-04]

duration: 12min
completed: 2026-04-02
---

# Phase 02 Plan 01: Review Gate & Global Routing Summary

**Stop hook review gate with ALLOW/BLOCK protocol, task-type-aware depth (D-12), and global opt-out routing replacing Phase 1 opt-in**

## Performance

- **Duration:** 12 min
- **Started:** 2026-04-02T19:35:00Z
- **Completed:** 2026-04-02T19:47:55Z
- **Tasks:** 2
- **Files modified:** 3 (1 created, 2 modified)

## Accomplishments

- Stop hook review gate created (429 lines): gates every Claude turn with code changes on a Codex ALLOW/BLOCK review before the user sees anything
- Task-type classification implemented per D-12: security patterns detected first, then test-gen (all files match), then bulk-ops (>10 files or all style/data), defaulting to feature — drives deep vs light review depth
- Global opt-out routing replacing Phase 1 opt-in: codex-router.js now fires for all projects with a `.claude/settings.json` unless `routing_disabled: true`, with backward compat for `routing_enabled: false` projects
- REVW-02 bidirectional review verified: Codex-reviews-Claude via Stop hook (new), Claude-reviews-Codex via codex-token-logger.js additionalContext injection (Phase 1, confirmed still present at line 96)
- Stop hook registered in `~/.claude/settings.json` with 300s timeout

## Task Commits

Each task committed atomically (hook files outside repo — tracked in planning artifacts):

1. **Task 1: Create codex-review-gate.js Stop hook** - `feat(02-01)` — codex-review-gate.js created, 429 lines, all 22 acceptance criteria passing
2. **Task 2: Extend codex-router.js + register Stop hook** - `feat(02-01)` — codex-router.js extended to v2.0.0 global routing, Stop hook registered in settings.json, REVW-02 verified

**Plan metadata:** Included in final commit with SUMMARY.md + STATE.md update

## Files Created/Modified

- `/home/alucard/.claude/hooks/codex-review-gate.js` (429 lines, created) — Stop hook with stop_hook_active guard, code change detection, task classification, depth-varied Codex review, ALLOW/BLOCK verdict parsing, token logging
- `/home/alucard/.claude/hooks/codex-router.js` (112 lines, modified from 102) — Updated to v2.0.0, global opt-out routing replacing Phase 1 opt-in
- `/home/alucard/.claude/settings.json` (modified) — Stop event key added with codex-review-gate.js hook and 300s timeout

## Hook Registration State After This Plan

```
Stop:
  1. codex-review-gate.js         timeout=300  [NEW - review gate]

PreToolUse (Write|Edit):
  1. claude-settings-guard.js    timeout=5  [existing - security guard]
  2. gsd-prompt-guard.js         timeout=5  [existing - workflow guard]
  3. codex-router.js             timeout=5  [existing v2.0.0 - global routing]

PostToolUse (Bash|Edit|Write|MultiEdit|Agent|Task):
  1. gsd-context-monitor.js      timeout=10 [existing - GSD context]
  2. codex-token-logger.js       timeout=10 [existing - token tracking]

SessionStart:
  1. gsd-check-update.js                    [existing - GSD updates]
```

## Decisions Made

- **Fail-open on all errors:** `process.exit(0)` in the outer catch block ensures the hook never blocks the user when Codex is unavailable, erroring, or timing out. Correctness matters more than enforcing review when the reviewer is down.
- **Phase 2 routing is opt-out not opt-in:** The `routing_disabled: true` opt-out pattern is more developer-friendly — projects benefit from routing advice by default once they have a `.claude/settings.json`. Backward compat maintained for `routing_enabled: false` projects via explicit check.
- **Stop hook 300s timeout:** Codex review can take 30-120 seconds depending on diff size and model load. The 120s `runCodexExec` timeout sits within the 300s hook timeout, leaving 180s for startup/teardown overhead.

## Deviations from Plan

None — plan executed exactly as written. All acceptance criteria met on first attempt. No bugs found, no blocking issues, no architectural changes needed.

## Issues Encountered

- Hook files (`~/.claude/hooks/`) are outside the project git repo at `/home/alucard/projects/Claude_X_Codex`. This matches Phase 1's pattern — hook files are created on disk and tracked via planning artifacts only. Per-task commits in the project repo contain only `.planning/` changes.

## REVW-02 Bidirectional Review Status

| Direction | Mechanism | Status |
|-----------|-----------|--------|
| Codex reviews Claude | Stop hook (`codex-review-gate.js`) | IMPLEMENTED (this plan) |
| Claude reviews Codex | `codex-token-logger.js` `additionalContext` injection at line 96 | VERIFIED PRESENT (Phase 1) |

Both directions confirmed operational. REVW-02 fully satisfied.

## Next Phase Readiness

- Plan 02-02 (GSD wave-boundary hook) can proceed — it builds on the same `codex-exec.js` infrastructure
- The Stop hook is live and will gate all code-modifying Claude turns from this point forward
- Concern from STATE.md: Phase 2 GSD wave state schema field names in `.planning/STATE.md` not verified from source — remains relevant for 02-02 planning

---
*Phase: 02-review-gate-gsd-integration*
*Completed: 2026-04-02*

## Self-Check: PASSED

| Item | Status |
|------|--------|
| `/home/alucard/.claude/hooks/codex-review-gate.js` exists | FOUND (429 lines) |
| `/home/alucard/.claude/hooks/codex-router.js` updated to v2.0.0 | FOUND (112 lines) |
| `/home/alucard/.claude/settings.json` has Stop hook with timeout 300 | FOUND |
| `codex-review-gate.js` node -c syntax check | PASSED |
| `codex-router.js` node -c syntax check | PASSED |
| `settings.json` valid JSON | PASSED |
| All 12 plan verification checks | 12/12 PASSED |
| `02-01-SUMMARY.md` exists | FOUND |
