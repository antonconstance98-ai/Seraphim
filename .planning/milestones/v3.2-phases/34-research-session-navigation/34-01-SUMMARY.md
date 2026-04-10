---
phase: 34-research-session-navigation
plan: "01"
subsystem: research
tags: [research-tracker, commands, interrogation-gate, crud]
dependency_graph:
  requires: []
  provides: [lib/research-tracker.js, commands/research-scope.md, commands/research-run.md]
  affects: [research-run.md, any future research commands]
tech_stack:
  added: []
  patterns: [atomic-tmp-rename-write, RSRCH-NNN-id-generation, two-command-gate]
key_files:
  created:
    - ~/.claude/plugins/seraphim/lib/research-tracker.js
    - ~/.claude/plugins/seraphim/commands/research-scope.md
    - ~/.claude/plugins/seraphim/commands/research-run.md
  modified: []
decisions:
  - "research-tracker.js follows requirements.js atomic tmp+rename pattern exactly"
  - "Two-command gate enforced: research-run.md aborts if getScopedItems returns empty array"
  - "RSRCH-NNN IDs zero-padded to 3 digits, matching existing REQ-NNN convention"
metrics:
  duration: "8 minutes"
  completed: "2026-04-10"
  tasks_completed: 2
  files_created: 3
---

# Phase 34 Plan 01: Research Tracker Lib + Commands Summary

**One-liner:** Atomic CRUD lib for research.json with RSRCH-NNN IDs, plus two-command scope/run separation enforcing mandatory human interrogation gate.

## What Was Built

### Task 1: lib/research-tracker.js

Created `/home/alucardmessangeroflight/.claude/plugins/seraphim/lib/research-tracker.js` with 6 exported functions:

- `readResearch(projectRoot)` — reads `.seraphim/research.json`, returns `{ items: {} }` on missing/error
- `writeResearch(projectRoot, data)` — atomic tmp+rename write, creates `.seraphim/` if missing
- `nextResearchId(data)` — scans RSRCH-NNN keys, returns next zero-padded ID
- `addResearch(projectRoot, { topic, questions, constraints, categories })` — creates scoped item
- `updateResearch(projectRoot, id, updates)` — merges updates, returns null if ID not found
- `getScopedItems(projectRoot)` — filters items by `status === "scoped"`

Pattern source: `requirements.js` (identical atomic read/write structure, parallel ID generation logic).

### Task 2: research-scope.md + research-run.md

**research-scope.md** conducts a 4-question interrogation gate:
1. Topic (if not in arguments)
2. Specific questions to answer
3. Constraints (what NOT to research)
4. Categories/tags

After user answers all questions, writes to `.seraphim/research.json` via `addResearch`.

**research-run.md** enforces the two-command separation:
- Calls `getScopedItems` — if empty, aborts with clear error directing user to run `research-scope` first
- Updates status to `running` before starting
- Conducts structured research per question in scope
- Writes results back with `status: complete` and `completed_at` timestamp

## Deviations from Plan

None — plan executed exactly as written.

## Commits

- `16add13` — feat(34-01): create lib/research-tracker.js with atomic CRUD
- `ef1b6d7` — feat(34-01): create research-scope.md and research-run.md commands

## Self-Check: PASSED

- research-tracker.js exists at expected path
- nextResearchId({ items: {} }) returns "RSRCH-001" (verified)
- research-scope.md has description frontmatter and addResearch call
- research-run.md has getScopedItems gate, "No scoped research found" error, and research-scope reference
