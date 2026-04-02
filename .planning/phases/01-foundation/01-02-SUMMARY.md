---
phase: 01-foundation
plan: 02
subsystem: infra
tags: [codex, hooks, token-tracking, execution-wrapper, jsonl]

requires:
  - 01-01 (AGENTS.md and .claude/settings.json)
provides:
  - codex-exec.js shared module (runCodexExec, parseCodexTokens, computeCost)
  - codex-token-logger.js PostToolUse hook (CODEX_RESULT marker detection, JSONL append)
  - .planning/token-log.jsonl append-only token tracking log
affects:
  - 01-03 (plan-review loop — will require runCodexExec from codex-exec.js)
  - 01-04 (token reporting — reads token-log.jsonl)
  - All future hooks that invoke Codex (require('./codex-exec'))

tech-stack:
  added: []
  patterns:
    - "codex-exec.js shared module: all hooks require() this instead of spawning Codex directly"
    - "CODEX_RESULT marker: hooks signal token data back to token logger via tool_result prefix"
    - "spawn not exec: streams JSONL output instead of buffering; prevents memory issues on long Codex runs"
    - "Last token_count event: parseCodexTokens takes the final token_count with non-null info (cumulative total)"
    - "300s SIGTERM + 5s SIGKILL: two-phase timeout kill prevents Codex CLI hang (Pitfall 1)"
    - ".planning/token-log.jsonl gitignored: session-specific data not committed"

key-files:
  created:
    - /home/alucard/.claude/hooks/codex-exec.js
    - /home/alucard/.claude/hooks/codex-token-logger.js
    - /home/alucard/projects/Claude_X_Codex/.planning/token-log.jsonl
    - /home/alucard/projects/Claude_X_Codex/.gitignore
  modified: []

key-decisions:
  - "spawn over exec: exec buffers all output; spawn streams JSONL line-by-line enabling real-time token extraction"
  - "Last token_count event: first token_count event has null info (Pitfall 3 from research); taking last ensures cumulative totals"
  - "CODEX_RESULT marker pattern: token logger detects Codex calls via tool_result prefix rather than tool_name — allows any tool to carry Codex results back"
  - "computeCost hardcoded pricing: dated comment (2026-04-02) so staleness is visible; no external pricing API dependency"

requirements-completed:
  - FNDTN-02
  - TRCK-01
  - TRCK-02

duration: ~5 min
completed: 2026-04-02
---

# Phase 01 Plan 02: Codex Execution Wrapper and Token Tracking Infrastructure Summary

**Codex exec wrapper with 300s timeout + SIGTERM/SIGKILL and JSONL token logger with CODEX_RESULT marker detection appending to append-only token-log.jsonl**

## Performance

- **Duration:** ~5 min
- **Started:** 2026-04-02T17:44:30Z
- **Completed:** 2026-04-02T17:49:27Z
- **Tasks:** 2 of 2 complete
- **Files created:** 4 (2 hooks at ~/.claude/hooks/, 1 log file, 1 .gitignore)

## Accomplishments

- Created codex-exec.js shared module with three exports: runCodexExec (300s timeout spawn), parseCodexTokens (last non-null token_count event), computeCost (hardcoded GPT-5.4 / GPT-5.4-mini pricing)
- Created codex-token-logger.js PostToolUse hook that detects [CODEX_RESULT] marker in tool_result, extracts token data, appends JSONL record to .planning/token-log.jsonl, and outputs advisory context
- Created empty .planning/token-log.jsonl (append-only log, gitignored)
- Created .gitignore excluding token-log.jsonl, node_modules, .env files
- All automated verifications pass: exports correct, parseCodexTokens parses verified JSONL schema, computeCost returns 0.0035 for 1000 input + 100 output tokens at gpt-5.4 pricing

## Task Commits

Each task was committed atomically:

1. **Task 1: Create codex-exec.js shared execution wrapper** — hook installed to ~/.claude/hooks/ (outside project repo; no git commit for hook file itself)
2. **Task 2: Create codex-token-logger.js and initialize token log** — `7d88a9e` (chore: .gitignore + token-log.jsonl)

Note: ~/.claude/hooks/ is not a git repository. Hook files are runtime-installed infrastructure. The project repo tracks planning state and project-side files (.gitignore, token-log.jsonl).

## Files Created/Modified

- `/home/alucard/.claude/hooks/codex-exec.js` — Shared Codex execution wrapper: runCodexExec, parseCodexTokens, computeCost; 300s SIGTERM+SIGKILL timeout; spawn-based streaming; no API key logging
- `/home/alucard/.claude/hooks/codex-token-logger.js` — PostToolUse hook: stdin timeout guard, CODEX_RESULT marker detection, JSONL append to token-log.jsonl, advisory context output; silent fail on errors
- `/home/alucard/projects/Claude_X_Codex/.planning/token-log.jsonl` — Empty append-only token log (gitignored)
- `/home/alucard/projects/Claude_X_Codex/.gitignore` — Excludes token-log.jsonl, node_modules, .env files

## Decisions Made

- spawn over exec: exec buffers all output; spawn streams JSONL line-by-line enabling real-time token extraction without memory issues on long Codex runs
- Last token_count event: first token_count event has null info (Pitfall 3 from research); parseCodexTokens takes the LAST event with non-null info to get cumulative totals
- CODEX_RESULT marker pattern: token logger detects Codex calls via tool_result prefix rather than tool_name — this allows any hook (PostToolUse, Stop, etc.) to carry Codex results back to the logger
- computeCost hardcoded pricing: includes dated comment (2026-04-02) so staleness is visible; avoids external pricing API dependency

## Deviations from Plan

None — plan executed exactly as written.

## Issues Encountered

- ~/.claude/hooks/ directory is not a git repository — hook files are installed as runtime infrastructure and tracked via planning state only. The plan's file_modified list references these paths but they cannot be git-committed from the project repo. This is the correct behavior.

## Known Stubs

None — no placeholder values, hardcoded empty returns, or TODO stubs in the created files. All three exported functions are fully implemented and verified.

## Next Phase Readiness

- codex-exec.js is ready to be required() by any hook in Phase 02 (routing hook) and Phase 03 (plan review loop)
- codex-token-logger.js is ready to be registered in ~/.claude/settings.json (Plan 03 registers all hooks)
- token-log.jsonl schema established: timestamp, session_id, model, source, task_type, tokens (input/cached_input/output/reasoning_output), cost_usd, rate_limit_pct
- .gitignore in place — token data will never be accidentally committed

---
*Phase: 01-foundation*
*Completed: 2026-04-02*
