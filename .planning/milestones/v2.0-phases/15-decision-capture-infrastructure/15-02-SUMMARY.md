---
phase: 15-decision-capture-infrastructure
plan: "02"
subsystem: decision-capture
tags: [gsd-commands, adaptive-control, dismiss-feedback, freeze-unfreeze, training-signal]
dependency_graph:
  requires: [decision-log.jsonl, hook-signal-state]
  provides: [gsd:dismiss-last, gsd:freeze, gsd:unfreeze, adaptive-flag-in-settings]
  affects: [.claude/settings.json, .planning/decision-log.jsonl]
tech_stack:
  added: []
  patterns:
    - GSD inline slash command pattern (no Task/Agent tools, process block only)
    - atomic write via writeFileSync-to-tmp + renameSync (project settings)
    - append-only JSONL for dismiss records (ref_timestamp links to original BLOCK)
    - 30-day rolling window for suppression threshold counting
key_files:
  created:
    - ~/.claude/commands/gsd/dismiss-last.md
    - ~/.claude/commands/gsd/freeze.md
    - ~/.claude/commands/gsd/unfreeze.md
  modified: []
decisions:
  - "/gsd:dismiss-last targets event_type===Stop guard in addition to review_verdict===BLOCK — prevents accidentally dismissing PostToolUse scan records that share the BLOCK verdict field"
  - "Dismiss record uses ref_timestamp field to link back to original BLOCK — Phase 16 noise profiler can join dismiss records to their source events without scanning entire log"
  - "freeze/unfreeze write to project-scope .claude/settings.json via process.cwd() — never ~/.claude/settings.json which would globally freeze all projects"
  - "adaptive flag lives at root level of settings.json (sibling of codex and minimax) — consistent with Phase 15 CONTEXT.md D-07 spec; hooks check settings.adaptive directly"
metrics:
  duration: 2 min
  completed: "2026-04-04"
  tasks_completed: 2
  files_created: 3
  files_modified: 0
---

# Phase 15 Plan 02: User Control Commands Summary

Three GSD slash commands giving the user direct control over the adaptive learning system: dismiss-last for false-positive BLOCK feedback with 30-day suppression tracking, freeze to revert to static v2.0 rules instantly, and unfreeze to re-enable adaptive behavior.

## Tasks Completed

| Task | Name | Commit | Key Files |
|------|------|--------|-----------|
| 1 | Create /gsd:dismiss-last command | 8611409 | ~/.claude/commands/gsd/dismiss-last.md |
| 2 | Create /gsd:freeze and /gsd:unfreeze commands | 218dcac | ~/.claude/commands/gsd/freeze.md, ~/.claude/commands/gsd/unfreeze.md |

## What Was Built

### dismiss-last.md (new)

GSD slash command for dismissing false-positive review BLOCKs.

- **Target selection:** Last record where `review_verdict === "BLOCK"` AND `outcome === null` AND `event_type === "Stop"`. The `event_type` guard prevents accidentally targeting PostToolUse scan records.
- **Append-only:** Writes a new dismiss record to `.planning/decision-log.jsonl` with `ref_timestamp` pointing to the original BLOCK — never modifies existing records.
- **Feedback loop:** Counts prior dismissals for the same `review_block_category` in the last 30 days, shows progress toward the 3-dismissal suppression threshold with three distinct messages (first/second/third+ dismissal).
- **Inline workflow:** No Task, no Agent. Read + Grep to scan log, Bash node one-liner to append, direct output for feedback.

Dismiss record schema:
```json
{
  "schema_version": 1,
  "timestamp": "<now>",
  "session_id": "<from original BLOCK>",
  "ref_timestamp": "<original BLOCK timestamp>",
  "event_type": "dismiss",
  "review_block_category": "<from original BLOCK>",
  "outcome": "dismissed",
  "dismissed_at": "<now>",
  "project": "<from original BLOCK>",
  "cwd": "<current>"
}
```

### freeze.md (new)

GSD slash command to freeze adaptive behavior for the current project.

- Reads `{cwd}/.claude/settings.json`, sets `adaptive: false` at root level, writes atomically via tmp+rename.
- Confirmation: "Adaptive behavior frozen. System will use static v2.0 rules. Run `/gsd:unfreeze` to re-enable."
- Scope guard documented: writes ONLY to project-scope file via `process.cwd()`, never to `~/.claude/settings.json`.

### unfreeze.md (new)

GSD slash command to re-enable adaptive behavior.

- Reads `{cwd}/.claude/settings.json`, sets `adaptive: true` at root level, writes atomically.
- Confirmation: "Adaptive behavior re-enabled. System will resume learning from decisions."
- Same scope guard as freeze.md.

## Decisions Made

1. **`event_type === "Stop"` guard on dismiss-last** — BLOCK events from `codex-review-gate.js` are always Stop-type events. Adding this guard makes the targeting specification self-documenting and prevents future confusion if a PostToolUse hook ever writes a field that looks like a review verdict.

2. **`ref_timestamp` for dismiss linkage** — Using the original BLOCK's timestamp as a foreign key means Phase 16's noise profiler can do a direct lookup join without scanning. More robust than storing only a line number (which would shift if the log were ever archived/rotated).

3. **30-day rolling window for suppression counting** — Matches the research recommendation. Prevents a rule from staying suppressed forever due to a cluster of dismissals from a single unusual project context months ago.

4. **`process.cwd()` for settings path in freeze/unfreeze** — Resolves to the project directory at runtime, which is where Claude Code sets cwd. Avoids hardcoding paths. The `NEVER ~/.claude/settings.json` warning is explicit in both files to prevent future drift.

## Deviations from Plan

None — plan executed exactly as written.

## Known Stubs

None. All three commands are fully specified inline workflows. The `outcome` and `dismissed_at` fields they write to decision-log.jsonl are the same fields that `decision-logger.js` leaves as `null` by design — these commands provide the mechanism to populate them when the user explicitly dismisses a false positive.

## Self-Check: PASSED

- FOUND: /home/alucard/.claude/commands/gsd/dismiss-last.md
- FOUND: /home/alucard/.claude/commands/gsd/freeze.md
- FOUND: /home/alucard/.claude/commands/gsd/unfreeze.md
- FOUND: commit 8611409 (feat(15-02): create /gsd:dismiss-last command)
- FOUND: commit 218dcac (feat(15-02): create /gsd:freeze and /gsd:unfreeze commands)
