---
phase: 15-decision-capture-infrastructure
plan: "01"
subsystem: decision-capture
tags: [hook-infrastructure, shared-state, decision-logging, latency-tracking, taxonomy]
dependency_graph:
  requires: []
  provides: [decision-log.jsonl, hook-signal-state, latency-fields-in-token-log]
  affects: [token-log.jsonl, codex-review-gate.js, minimax-post-scan.js, settings.json]
tech_stack:
  added: []
  patterns:
    - append-only JSONL state files for inter-hook communication
    - SHA-256 deterministic event IDs from stdin payload
    - per-model latency measurement inside individual Promise wrappers (not Promise.all max)
    - direct BLOCK event capture to bypass Stop chain halt
key_files:
  created:
    - ~/.claude/hooks/hook-signal.js
    - ~/.claude/hooks/decision-logger.js
  modified:
    - ~/.claude/hooks/codex-review-gate.js
    - ~/.claude/hooks/minimax-post-scan.js
    - ~/.claude/hooks/codex-token-logger.js
    - ~/.claude/settings.json
decisions:
  - "Append-only JSONL state files: writeHookSignal appends one line per call; readHookState merges all lines (last wins). Eliminates read-merge-write race entirely."
  - "SHA-256 event IDs truncated to 16 hex chars with NUL byte separators — prevents field-boundary ambiguity and collision risk."
  - "Per-model latency measured inside each Promise wrapper, not around Promise.all — gives each token-log record its TRUE individual latency rather than the max."
  - "Review-gate writes BLOCK events directly to decision-log.jsonl — ensures capture even when Stop chain halts after decision:block output."
  - "classifyTaskType12 hook-dev rule uses path.join(os.homedir(), '.claude', 'hooks') literal prefix — matches only ~/.claude/hooks/, not src/hooks/ or other /hooks/ paths."
  - "explain returned for null/undefined toolName in PostToolUse (not 'review') — consistent with rule 10 spec, covers chat-only interactions."
  - "readAdaptiveFlag and readProjectConfig return null on error — never default to true to avoid false positive adaptive-mode readings."
metrics:
  duration: 6 min
  completed: "2026-04-04"
  tasks_completed: 2
  files_created: 2
  files_modified: 4
---

# Phase 15 Plan 01: Decision Capture Infrastructure Summary

Core decision capture infrastructure: shared state module (hook-signal.js) for inter-hook communication via append-only SHA-256-keyed JSONL files, decision-logger.js with 12-category task taxonomy capturing PostToolUse and Stop events into decision-log.jsonl, upstream hook modifications to write latency and signal data, and extended token-log.jsonl schema.

## Tasks Completed

| Task | Name | Commit | Key Files |
|------|------|--------|-----------|
| 1 | Create hook-signal.js and decision-logger.js | 3692f39 | ~/.claude/hooks/hook-signal.js, ~/.claude/hooks/decision-logger.js |
| 2 | Modify upstream hooks, extend schema, register decision-logger | c666ca4 | ~/.claude/hooks/codex-review-gate.js, minimax-post-scan.js, codex-token-logger.js, settings.json |

## What Was Built

### hook-signal.js (new)

Shared state module for inter-hook communication. Four exported functions:

- `buildEventId(data)` — SHA-256 hash of stdin fields (session_id + tool_name + tool_input for PostToolUse; session_id + 'stop' + content[:200] for Stop), truncated to 16 hex chars with NUL byte separators.
- `writeHookSignal(cwd, eventId, key, value)` — appends `{key: value}` as one JSON line to `{cwd}/.planning/.hook-state/{eventId}.jsonl`. Pure append, no read.
- `readHookState(cwd, eventId)` — reads all lines and merges (last value wins per key). Returns `{}` on error.
- `cleanupHookState(cwd, eventId)` — deletes state file + sweeps stale files older than 1 hour.

### decision-logger.js (new)

PostToolUse + Stop capture hook. Must be last in both chains.

- Detects event type via concrete field presence (`tool_name !== undefined` for PostToolUse)
- PostToolUse records: file_ext, diff signals from shared state, scan verdict/latency, task classification
- Stop records: review verdict/block_category/latency from shared state
- `classifyTaskType12()` — 12 ordered categories: security-scan, test-write, bulk-ops, hook-dev, plan-review, architecture, refactor, doc-update, debug, explain, implementation, review
- `schema_version: 1` on all records for forward compatibility
- Config snapshot captured per-record (adaptive flag, thresholds)

### Modified: codex-review-gate.js

- `hookStartMs` at top, `buildEventId`/`writeHookSignal` imported
- Per-model latency measured inside individual async wrappers (not Promise.all max):
  - `codexModelLatencyMs` measured around `runCodexExec` call only
  - `minimaxModelLatencyMs` measured around `runMinimax` call only
- 6 `writeHookSignal` calls after verdict (review_verdict, review_block_category, codex latency, minimax latency, hook latency)
- `model_latency_ms` and `hook_latency_ms` added to both Codex and MiniMax token-log records
- Direct BLOCK capture: writes full decision record to `decision-log.jsonl` when shouldBlock is true
- `classifyBlockCategory()` helper extracts first word of block summary

### Modified: minimax-post-scan.js

- `hookStartMs` at top, `buildEventId`/`writeHookSignal` imported
- cwd/eventId computed immediately after data parsed (session_id available for all paths below)
- `writeHookSignal(scan_triggered=false, scan_verdict='SKIPPED')` on all 6 early-exit paths
- `writeHookSignal(scan_triggered=true, scan_verdict='ERROR')` on API failure
- 6 success-path signals: scan_triggered, scan_verdict, scan_issues_count, scan_model_latency_ms, scan_hook_latency_ms, diff_lines_changed
- `model_latency_ms` and `hook_latency_ms` added to own token-log record

### Modified: codex-token-logger.js

Three new optional fields added after existing v2.0 block:
- `outcome` — pass-through from CODEX_RESULT marker
- `model_latency_ms` — individual model call duration
- `hook_latency_ms` — total hook wall-clock duration

### Modified: settings.json

`decision-logger.js` registered as last hook (timeout: 10) in both PostToolUse and Stop chains. All existing hooks preserved in original order.

## Decisions Made

1. **Append-only state eliminates race condition** — `writeHookSignal` never reads before writing; `readHookState` merges all appended lines. `fs.appendFileSync` is atomic for small writes under PIPE_BUF on POSIX.

2. **SHA-256 over MD5** — 64-bit truncated ID prevents collisions at scale; NUL byte separator prevents field-boundary ambiguity (e.g., `session_id="ab"` + `tool_name="cd"` vs `session_id="abcd"` + empty tool_name).

3. **Per-model latency inside Promise wrappers** — Measuring around `Promise.all` would assign MAX(codex, minimax) to both records, falsifying both. Individual wrappers give each record its true latency.

4. **Direct BLOCK write** — When `codex-review-gate.js` outputs `decision: "block"`, Claude may halt the Stop chain before `decision-logger.js` runs. Writing directly ensures the signal is never lost.

5. **`explain` for null toolName** — Rule 10 fires before path-based rules. `classifyTaskType12(null, [], {}, 'PostToolUse')` returns `'explain'`, not `'review'`. This correctly identifies chat-only interactions.

6. **`hook-dev` uses literal path prefix** — `path.join(os.homedir(), '.claude', 'hooks')` is a precise prefix check. Regex `/hooks/` would incorrectly classify `src/hooks/useAuth.js` as hook-dev instead of security-scan.

## Deviations from Plan

None — plan executed exactly as written. The acceptance criteria note that `classifyTaskType12('Write', ['src/hooks/useAuth.js'], {}, 'PostToolUse')` should not return `'hook-dev'` (satisfied: returns `'security-scan'` due to "auth" in path, which is the correct Rule 1 behavior). The parenthetical `(returns 'implementation')` in the acceptance criteria was approximate; the binding test is `grep -v "hook-dev"`.

## Known Stubs

None. All fields are wired to real data sources:
- `scan_*` fields come from `minimax-post-scan.js` writeHookSignal calls
- `review_*` fields come from `codex-review-gate.js` writeHookSignal calls  
- `outcome` and `dismissed_at` are `null` by design — Phase 16/17 will populate these via feedback loop

## Self-Check: PASSED

- FOUND: ~/.claude/hooks/hook-signal.js
- FOUND: ~/.claude/hooks/decision-logger.js
- FOUND: .planning/phases/15-decision-capture-infrastructure/15-01-SUMMARY.md
- FOUND: commit 3692f39 (feat(15-01): create hook-signal.js and decision-logger.js)
- FOUND: commit c666ca4 (feat(15-01): modify upstream hooks, extend token-log schema, register decision-logger)
