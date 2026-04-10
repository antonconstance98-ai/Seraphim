---
phase: 33-core-command-layer
plan: "02"
subsystem: commands
tags: [seed-capture, note, todo, promote, idea-funnel]
dependency_graph:
  requires: ["33-01"]
  provides: [seed.md, note.md, promote.md, add-todo.md, check-todos.md]
  affects: [idea-capture-funnel, feature-promotion-flow, todo-tracking]
tech_stack:
  added: []
  patterns: [yaml-frontmatter-command, bash-walk-up-root-discovery, node-e-lib-require, jsonl-append]
key_files:
  created:
    - ~/.claude/plugins/seraphim/commands/seed.md
    - ~/.claude/plugins/seraphim/commands/note.md
    - ~/.claude/plugins/seraphim/commands/promote.md
    - ~/.claude/plugins/seraphim/commands/add-todo.md
    - ~/.claude/plugins/seraphim/commands/check-todos.md
  modified: []
decisions:
  - "note.md has zero interaction by design — $ARGUMENTS is used verbatim as the note body, no questions asked (SEED-05)"
  - "promote.md stages requirement approval interactively before writing — user confirms before any writes"
  - "todos.jsonl is JSONL append-only; check-todos rewrites entire file only on --done (idiomatic for small lists)"
metrics:
  duration_minutes: 12
  completed_date: "2026-04-10"
  tasks_completed: 2
  files_created: 5
  files_modified: 0
---

# Phase 33 Plan 02: Core Command Layer (Seed/Note/Todo) Summary

**One-liner:** Five idea-capture commands — seed, note, promote, add-todo, check-todos — wiring the idea-to-shipped funnel from raw braindump to tracked feature.

## Objective

Create the command files that let users capture ideas (seed.md), fire off zero-friction notes (note.md), promote seeds to roadmap features with requirements (promote.md), and manage lightweight todos (add-todo.md, check-todos.md).

## What Was Built

| Command | File | Purpose |
|---------|------|---------|
| `/seraphim:seed` | commands/seed.md | Captures a raw idea with title, tags, freeform body via seed-store.js writeSeed |
| `/seraphim:note` | commands/note.md | Zero-friction note — no questions, writes NOTE-NNN.md instantly |
| `/seraphim:promote` | commands/promote.md | Promotes SEED-NNN to a feature: suggests requirements, writes via requirements.js, adds to roadmap.js |
| `/seraphim:add-todo` | commands/add-todo.md | Appends TODO-NNN to .planning/todos.jsonl with area tagging |
| `/seraphim:check-todos` | commands/check-todos.md | Lists pending todos grouped by area; supports --done and --area flags |

## Tasks

| # | Task | Commit | Status |
|---|------|--------|--------|
| 1 | Create seed.md, note.md, promote.md | b24e8d8 | Done |
| 2 | Create add-todo.md, check-todos.md | 17b4f65 | Done |

## Deviations from Plan

None — plan executed exactly as written.

## Known Stubs

None. All commands are fully specified with concrete lib references and data paths. No placeholder text or hardcoded empty values in the instruction flow.

## Self-Check: PASSED

- seed.md: exists, contains `description:`, contains `seed-store` reference, contains bash walk-up pattern
- note.md: exists, contains `description:`, zero interaction (no ask/question lines), writes to .planning/notes/NOTE-NNN.md
- promote.md: exists, contains `description:`, references seed-store.js, requirements.js, and roadmap.js
- add-todo.md: exists, contains `description:`, writes to .planning/todos.jsonl, supports --area flag
- check-todos.md: exists, contains `description:`, reads todos.jsonl, supports --done and --area flags, groups by area
