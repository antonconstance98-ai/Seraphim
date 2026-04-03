---
phase: 12-context-compression
plan: "01"
subsystem: infra
tags: [minimax, compression, hooks, posttooluse, node]

# Dependency graph
requires:
  - phase: 11-posttooluse-bug-scanner
    provides: minimax-post-scan.js structural pattern (stdin handling, stdinTimeout, lazy-require, advisory output, token logging)
  - phase: 08-minimax-foundation
    provides: minimax-exec.js runMinimax() function and codex-pricing.js computeCodexCostStrict()
provides:
  - minimax-compress.js dual-mode compression module (PostToolUse hook + require() library)
  - compress(text, opts) exported function for any hook to lazy-require
  - compress_context_pct, compress_tool_output_tokens, compress_diff_chars config keys in project settings
affects: [12-02-gsd-context-monitor, future review hooks using compress() for large diffs]

# Tech tracking
tech-stack:
  added: []
  patterns:
    - "Dual-mode module: single file runs as PostToolUse hook (require.main === module) AND exports library function"
    - "Purpose-aware compression prompts: 4 named purposes + generic fallback preserving all actionable/technical info"
    - "Lazy-require minimax-exec and codex-pricing only when compression fires (avoids SDK load on fast-exit paths)"
    - "Double-compression guard via .includes() (not .startsWith()) to catch header even after leading whitespace"
    - "tool_result coercion via JSON.stringify for non-string types (objects, arrays, null handled safely)"
    - "9500-char output truncation: 500-char buffer under 10K additionalContext ceiling"

key-files:
  created:
    - "~/.claude/hooks/minimax-compress.js — Dual-mode: PostToolUse hook + compress() library"
  modified:
    - ".claude/settings.json — Added compress_context_pct (60), compress_tool_output_tokens (10000), compress_diff_chars (8000)"

key-decisions:
  - "Dual-mode architecture: require.main === module guard keeps hook and library in one file (consistent with Phase 11 pattern)"
  - "Generic fallback for unknown purpose strings: preserves all actionable/technical info rather than silently compressing poorly"
  - "60s MiniMax timeout (not default 90s): accounts for up to 55s pre-answer latency while staying inside hook window"
  - "tool_result coercion covers null/undefined via != null check before JSON.stringify (avoids 'null' string output)"

patterns-established:
  - "Dual-mode module pattern: require.main === module guard for hook/library split"
  - "Purpose-aware compression: getPurposeRules() switch with named cases and generic fallback"

requirements-completed: [COMPRESS-CORE]

# Metrics
duration: 2min
completed: 2026-04-03
---

# Phase 12 Plan 01: Context Compression Core Summary

**MiniMax compress() library + PostToolUse hook compressing large tool outputs above a configurable token threshold, with purpose-aware prompts and fail-silent advisory output**

## Performance

- **Duration:** 2 min
- **Started:** 2026-04-03T20:38:02Z
- **Completed:** 2026-04-03T20:40:21Z
- **Tasks:** 2
- **Files modified:** 2

## Accomplishments
- Created `minimax-compress.js` as the first dual-mode hook in this codebase — runs as a PostToolUse hook from stdin and also exports `compress(text, opts)` for any hook to `require()` and call directly
- Implemented purpose-aware compression prompts for 4 named purposes (summarize code diff, condense tool output, compress conversation, condense file content) plus a generic fallback that preserves all actionable/technical information
- Added three compression threshold config keys to project settings: `compress_context_pct` (60%), `compress_tool_output_tokens` (10000), `compress_diff_chars` (8000)

## Task Commits

Each task was committed atomically:

1. **Task 1: Create minimax-compress.js dual-mode module** - `3c71762` (feat)
2. **Task 2: Add compression thresholds to project settings** - `7ed8159` (feat)

**Plan metadata:** (docs commit follows)

## Files Created/Modified
- `~/.claude/hooks/minimax-compress.js` — Dual-mode compression module: PostToolUse hook + `compress()` library export
- `.claude/settings.json` — Three new compression threshold keys added to minimax config block

## Decisions Made
- Dual-mode architecture using `require.main === module` guard: keeps hook and library in one file, consistent with how Phase 11 organized `minimax-post-scan.js`. Only `compress()` is exported; `buildCompressionPrompt` and `runAsHook` are private.
- Generic fallback for unknown purpose strings explicitly preserves "all actionable and technical information, error messages, file paths, function names, data values, and status indicators" — safer than silently compressing with no guidance.
- `tool_result` coercion uses `!= null` check before `JSON.stringify` to avoid outputting the string `"null"` for null/undefined inputs.
- 60s MiniMax timeout (not the 90s default in minimax-exec): accounts for documented up to 55s pre-answer latency while keeping margin inside hook window.

## Deviations from Plan

None - plan executed exactly as written.

## Issues Encountered
None. The `~/.claude/hooks/` directory is outside the project git repo (established pattern from all prior phases). Hook file committed with an empty-commit note describing the installed file, consistent with Phase 11 commit `9cb3758`.

## User Setup Required
None — no external service configuration required. The PostToolUse hook is not yet registered in settings.json (Plan 02 handles hook wiring alongside `gsd-context-monitor.js`).

## Known Stubs
None — `compress()` is a fully functional API call, not stubbed. The hook is not yet wired to settings.json (that is Plan 02's scope, intentional — this plan delivers the library first).

## Next Phase Readiness
- `compress()` is ready to require() from any hook: `const { compress } = require('/home/alucard/.claude/hooks/minimax-compress');`
- Plan 02 (`gsd-context-monitor.js`) can use `compress_context_pct` from settings.json immediately
- Review hooks can call `compress()` for diffs exceeding `compress_diff_chars` (8000 chars)
- PostToolUse hook wiring and `gsd-context-monitor.js` creation are Plan 02's scope

---
*Phase: 12-context-compression*
*Completed: 2026-04-03*
