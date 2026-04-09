---
phase: 04-quality-gates-and-decision-logging
plan: 02
subsystem: forge-command
tags: [checkpoint-gate, retry-loop, crucible-fix-mode, QUAL-02, QUAL-04, QUAL-05]
dependency_graph:
  requires: [04-01]
  provides: [forge-checkpoint-gate, forge-retry-loop, crucible-fix-mode]
  affects: [forge.md, checkpoint.js, phase-state.js]
tech_stack:
  added: []
  patterns: [checkpoint-gate-after-task, retry-with-feedback-injection, crucible-fail-detection]
key_files:
  modified:
    - ~/.claude/plugins/seraphim/commands/forge.md
decisions:
  - "cfg.max_loops used as retry cap — no separate max_retries variable introduced"
  - "incrementRetry always called (never manual state mutation) — writes to disk on every increment for crash safety"
  - "Fix mode skips tasks not in crucible issues with dedicated status='skipped' marker"
  - "PHASE_COMPLETE marker conditionally includes fix_mode and fix_task_count attributes"
metrics:
  duration: 3min
  completed: "2026-04-08T22:36:06Z"
  tasks_completed: 2
  files_modified: 1
---

# Phase 04 Plan 02: Forge Checkpoint Gate, Retry Loop, and Crucible Fix Mode Summary

Wired checkpoint gate with retry-with-feedback loop and crucible fix mode detection into forge.md, replacing the Phase 3 stub at Step 6e.

## Tasks Completed

| Task | Name | Commit | Files |
|------|------|--------|-------|
| 1 | Add checkpoint gate and retry-with-feedback to forge.md Step 6 | e4c0887 | commands/forge.md |
| 2 | Add crucible fix mode detection to forge.md Step 1/2 | 8f27fb2 | commands/forge.md |

## What Was Built

**Task 1 — Checkpoint gate and retry loop (Step 6e):**

Replaced the Phase 3 stub with a complete implementation:
- After each task execution and forge-log.md entry (Step 6d), `runCheckpoint` is called with `projectRoot`, `phaseId`, `taskId`, `taskType`, and `pluginRoot`.
- On `passed === true`: continue to next task.
- On `passed === false`: call `phaseState.incrementRetry(projectRoot, phaseId, task.id)` to get the retry count (writes to disk atomically).
- If `retryCount > cfg.max_loops`: emit escalation message with "exceeded the retry cap", full findings, and three manual resolution options (re-run, fix blueprint, reset state.json). Stop forge entirely.
- If `retryCount <= cfg.max_loops`: build retry prompt with `## Checkpoint Findings (Retry N/M)` header, first 10 findings, and original task spec. Re-execute via same dispatch-or-inline path. Append retry entry to forge-log.md with `retry="N"` attribute on the SERAPHIM:FORGE_TASK marker.
- Dependency halt logic moved to new **Step 6g** (runs only after all retries exhausted).

Step renumbering after fix mode insertion: 6a=fix-mode-filter, 6b=branch-on-project-type, 6c=dispatch-or-inline, 6d=write-forge-log, 6e=no-auto-commit, 6f=checkpoint-gate-retry, 6g=dependency-halt.

**Task 2 — Crucible fix mode (Step 2b):**

New Step 2b inserted after blueprint parsing:
- Reads `crucible.md` if it exists for the phase.
- Calls `parseMarkers` and looks for `PHASE_COMPLETE` with `phase === 'crucible'` and `verdict === 'fail'`.
- On fail verdict: sets `fixMode = true`, extracts task IDs referenced in crucible prose via `/\btask[-\s](\S+)/gi`, extracts optional `## Fix Instructions` block.
- Safety guard: if `fixTaskIds.size === 0` after extraction, logs a warning and sets `fixMode = false` (avoids silently running nothing).

Step 6a added before project_type branching:
- If `fixMode && !fixTaskIds.has(task.id)`: append SKIPPED entry with `status="skipped" fix_mode="true"` and `continue`.
- If `fixMode && fixTaskIds.has(task.id)`: prepend `## Crucible Fix Instructions` block with global fix instructions (or fallback text) before original task spec.

Step 7 PHASE_COMPLETE marker updated to conditionally include `fix_mode="true" fix_task_count="{N}"` when fix mode was active.

## Deviations from Plan

None — plan executed exactly as written.

## Self-Check

- [x] forge.md Step 6f calls `runCheckpoint` after every task execution
- [x] forge.md uses `incrementRetry` (never manual state mutation) for retry counting
- [x] Retry prompt format includes `## Checkpoint Findings` header and original task spec
- [x] Escalation message mentions "exceeded the retry cap" with manual resolution steps
- [x] Step 2b detects `crucible.md` verdict=fail and sets `fixMode=true`
- [x] Fix mode skips tasks not in crucible issues with `status="skipped"` marker
- [x] Fix mode prepends fix instructions to targeted tasks
- [x] Commits exist: e4c0887 (Task 1), 8f27fb2 (Task 2)
