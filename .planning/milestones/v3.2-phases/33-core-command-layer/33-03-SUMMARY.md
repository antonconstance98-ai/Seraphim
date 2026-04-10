---
phase: 33-core-command-layer
plan: 03
subsystem: seraphim-commands
tags: [requirements, discuss, assumptions, commands, context]
dependency_graph:
  requires: [33-01]
  provides: [requirements-command, discuss-command, assumptions-command]
  affects: [planning-workflow, context-generation]
tech_stack:
  added: []
  patterns: [node-e-require, project-root-walk-up, gsd-context-format]
key_files:
  created:
    - ~/.claude/plugins/seraphim/commands/requirements.md
    - ~/.claude/plugins/seraphim/commands/discuss.md
    - ~/.claude/plugins/seraphim/commands/assumptions.md
  modified: []
decisions:
  - "requirements command uses AI suggest + human approve pattern — never auto-commits requirements"
  - "discuss command produces CONTEXT.md with exact GSD XML tags: <domain>, <decisions>, <code_context>, <specifics>, <deferred>"
  - "assumptions command is read-only by default — only writes to CONTEXT.md on explicit user request"
metrics:
  duration: 3min
  completed_date: "2026-04-10T01:33:49Z"
  tasks: 2
  files: 3
---

# Phase 33 Plan 03: Core Command Layer (Requirements/Discuss/Assumptions) Summary

**One-liner:** Three Seraphim commands bridging idea capture to planning — AI-suggested requirements with human approval, interactive decision-locking producing GSD-compatible CONTEXT.md, and implicit assumption enumeration for pre-planning review.

## Tasks Completed

| Task | Name | Commit | Files |
|------|------|--------|-------|
| 1 | Create requirements.md command | ddf2509 | commands/requirements.md |
| 2 | Create discuss.md and assumptions.md commands | b580b7a | commands/discuss.md, commands/assumptions.md |

## What Was Built

### `/seraphim:requirements`

Defines requirements with AI suggestions and human approval (per REQ-01, REQ-02, REQ-03):
- Reads existing requirements via `lib/requirements.js` before suggesting
- Suggests requirements grouped by category with `v1` / `future` / `out-of-scope` scope labels
- Presents all suggestions for user review — **no writes until user approves**
- `--matrix` flag displays full traceability matrix (REQ-ID, category, scope, phase, feature, status, verified)
- Writes approved requirements via `addRequirement` one at a time

### `/seraphim:discuss`

Locks implementation decisions before planning (per PLAN-04, D-08):
- Reads roadmap, requirements, RESEARCH.md, and existing CONTEXT.md for context
- Conducts interactive discussion — surfaces each decision, presents options with tradeoffs, waits for user input
- Numbers decisions sequentially as D-01, D-02, etc.
- Writes CONTEXT.md with **exact GSD format**: `<domain>`, `<decisions>`, `<code_context>`, `<specifics>`, `<deferred>`
- Output is compatible with GSD planning agents that read CONTEXT.md

### `/seraphim:assumptions`

Surfaces implicit assumptions for pre-planning review (per PLAN-05):
- Reads CONTEXT.md, RESEARCH.md, requirements, roadmap, and config for context
- Enumerates assumptions across 5 categories: Technical, Scope, Dependencies, Data, Integration
- Presents numbered list for user to confirm, challenge, or add to
- Read-only by default — only amends CONTEXT.md on explicit user request

## Deviations from Plan

None — plan executed exactly as written.

## Known Stubs

None. All commands are fully specified with complete step-by-step flows.

## Self-Check: PASSED

- `~/.claude/plugins/seraphim/commands/requirements.md` — exists, contains `description:`, `matrix`, requirements workflow
- `~/.claude/plugins/seraphim/commands/discuss.md` — exists, contains `<domain>`, `CONTEXT.md` reference, D-01 numbering
- `~/.claude/plugins/seraphim/commands/assumptions.md` — exists, contains `assumptions`, 5 category structure
- Commits ddf2509 and b580b7a exist in plugin repo
