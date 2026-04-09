---
phase: 13-codex-execution-pipeline
plan: "01"
subsystem: orchestration
tags: [codex-cli, minimax, fallback-chain, gsd-executor, handoff-spec, token-logging]

# Dependency graph
requires:
  - phase: 08-minimax-foundation
    provides: runMinimax, isCodexRateLimited, runWithFallback interfaces
  - phase: 09-dual-review-gate
    provides: codex-exec.js runCodexExec signature
  - phase: 11-posttooluse-bug-scanner
    provides: lazy-require pattern, think-tag stripping pattern
  - phase: 12-context-compression
    provides: file-content injection pattern for MiniMax API-only constraint
provides:
  - codex-handoff.js: executeHandoff() three-tier fallback chain (Codex CLI -> MiniMax API -> user prompt)
  - gsd-executor.md: thin orchestrator pattern that generates handoff specs instead of writing code
  - CODEX_RESULT marker emission for execution events (token logging compatibility)
  - minimaxText return path: executor writes MiniMax output to disk via its own Write tool
affects:
  - gsd-executor (all future plan executions use handoff spec pattern)
  - token-log.jsonl (execution events now logged via CODEX_RESULT marker)
  - daily spend (Opus code-writing cost replaced by Codex CLI subscription cost)

# Tech tracking
tech-stack:
  added: []
  patterns:
    - "Three-tier execution fallback: Codex CLI (primary) -> MiniMax API (rate-limit only) -> user prompt (fail-closed)"
    - "Handoff spec format: task description + file paths + action type + specification + verification command"
    - "Lazy-require inside function body: codex-exec, minimax-exec, codex-pricing not loaded at module top level"
    - "Think-tag stripping: find </think>, take content after it, trim — consistent with Phase 11 pattern"
    - "Token normalization: all four fields always present in CODEX_RESULT marker (reasoning_output_tokens=0 for MiniMax)"
    - "Allow-empty git commits for files installed outside the project repo (~/.claude/...)"

key-files:
  created:
    - "~/.claude/hooks/codex-handoff.js"
  modified:
    - "~/.claude/agents/gsd-executor.md"

key-decisions:
  - "MiniMax fallback only on rate-limit (not all Codex failures) — non-rate-limit failures indicate config problems (Pitfall 4 from research)"
  - "Non-rate-limit Codex failures skip MiniMax entirely and go directly to user prompt (fail-closed)"
  - "Partial success guard: empty MiniMax text is feature-level failure even when success:true (Pattern from Phase 10)"
  - "minimaxText returned to executor for Write tool call — MiniMax has no filesystem access, executor writes on its behalf (D-10)"
  - "executeHandoff is a separate module (not inline in gsd-executor) — reusable by future consumers, testable in isolation"
  - "Only execute_tasks step replaced in gsd-executor.md — all other protocol sections byte-for-byte identical (D-07)"

patterns-established:
  - "Handoff spec pattern: executor generates ~500-word natural language spec per task, delegates to Codex/MiniMax"
  - "Validation after handoff: git diff --stat + verify command; retry once on failure; defer on second failure"
  - "Multi-file task discretion: one executeHandoff per file unless truly inseparable"

requirements-completed: []

# Metrics
duration: 4min
completed: 2026-04-03
---

# Phase 13 Plan 01: Codex Execution Pipeline Summary

**codex-handoff.js three-tier fallback (Codex CLI free -> MiniMax API on rate-limit -> user prompt fail-closed) wired into gsd-executor.md thin orchestrator pattern replacing direct Opus code-writing**

## Performance

- **Duration:** ~4 min
- **Started:** 2026-04-03T21:04:21Z
- **Completed:** 2026-04-03T21:08:31Z
- **Tasks:** 2
- **Files modified:** 2 (both outside project repo at ~/.claude/)

## Accomplishments

- Created `codex-handoff.js` with `executeHandoff()` implementing Codex CLI -> MiniMax API -> user prompt chain
- MiniMax fallback correctly gates on rate-limit detection only (non-rate-limit failures skip directly to user prompt)
- Modified `gsd-executor.md` execute_tasks step to generate handoff specs and delegate code writing to Codex
- All existing GSD protocols preserved unchanged in gsd-executor.md (deviation rules, checkpoints, TDD, commits, SUMMARY, state updates)
- CODEX_RESULT markers emitted on success for token logging compatibility with codex-token-logger.js

## Task Commits

Each task was committed atomically:

1. **Task 1: Create codex-handoff.js execution helper** - `1658c65` (feat -- allow-empty, file outside repo)
2. **Task 2: Modify gsd-executor.md to use handoff spec pattern** - `00bb495` (feat -- allow-empty, file outside repo)

## Files Created/Modified

- `~/.claude/hooks/codex-handoff.js` - Three-tier fallback chain with executeHandoff() export, think-tag stripping, CODEX_RESULT marker emission, token normalization
- `~/.claude/agents/gsd-executor.md` - execute_tasks step replaced with handoff spec pattern; Phase 13 maintenance comment added

## Decisions Made

- **MiniMax fallback only on rate-limit:** Non-rate-limit Codex failures (auth errors, misconfigurations) skip MiniMax and go directly to user prompt. Sending to MiniMax when Codex has a config error wastes API credits without solving the underlying problem (Pitfall 4 from Phase 8 research).
- **Partial success guard:** Empty MiniMax text (`success:true` but `text.trim()===''`) treated as feature-level failure and falls through to user prompt. Pattern established in Phase 10 adversarial review.
- **minimaxText as return value:** MiniMax has no filesystem access (API-only). Rather than having codex-handoff.js write files, the executor receives `minimaxText` and writes it via its own Write tool — keeps file I/O in the orchestrator, not the helper module.
- **Surgical gsd-executor.md edit:** Only the execute_tasks step was modified. All other sections (15+) remain byte-for-byte identical per D-07. A comment was added after the step for future maintainability.
- **Allow-empty commits:** Both modified files live at `~/.claude/...` (outside the project repo). Consistent with prior phase commit pattern (Phase 11, 12) — allow-empty commits document the installation.

## Deviations from Plan

None - plan executed exactly as written.

## Issues Encountered

None.

## User Setup Required

None - no external service configuration required. The handoff chain uses existing MINIMAX_API_KEY environment variable and Codex CLI authentication already configured in Phase 8.

## Next Phase Readiness

- `codex-handoff.js` is ready for use by gsd-executor in all future plan executions
- `gsd-executor.md` thin orchestrator pattern active — next plan execution will route code writing to Codex CLI
- Phase 14 (three-model reporting) can proceed; no blockers from this plan
- Token logging of execution events will flow through existing codex-token-logger.js PostToolUse hook

---
*Phase: 13-codex-execution-pipeline*
*Completed: 2026-04-03*

## Self-Check: PASSED

- FOUND: ~/.claude/hooks/codex-handoff.js
- FOUND: ~/.claude/agents/gsd-executor.md
- FOUND: 13-01-SUMMARY.md
- FOUND: commit 1658c65 (Task 1: codex-handoff.js)
- FOUND: commit 00bb495 (Task 2: gsd-executor.md)
