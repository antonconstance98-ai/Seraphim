---
phase: 35-phase-management-config-ui-tooling
plan: "04"
subsystem: commands
tags: [ui-tooling, quality-audit, test-generation, commands]
dependency_graph:
  requires: []
  provides: [ui-spec-command, ui-review-command, add-tests-command]
  affects: [frontend-phases, quality-workflow]
tech_stack:
  added: []
  patterns: [seraphim-command-md, ascii-wireframe, 6-pillar-audit, tdd-skeleton]
key_files:
  created:
    - ~/.claude/plugins/seraphim/commands/ui-spec.md
    - ~/.claude/plugins/seraphim/commands/ui-review.md
    - ~/.claude/plugins/seraphim/commands/add-tests.md
  modified: []
decisions:
  - "ui-review guards non-UI phases with early exit (no audit report created if no .tsx/.jsx/.css/.scss/.html files found)"
  - "add-tests checks existing test files before generating to avoid duplicating covered criteria (Pitfall 7)"
  - "add-tests generates framework-agnostic skeletons with install note when neither vitest nor jest detected"
metrics:
  duration: "5 min"
  completed: "2026-04-10"
  tasks: 2
  files: 3
requirements: [UI-01, UI-02, UI-03]
---

# Phase 35 Plan 04: UI and Quality Tooling Commands Summary

**One-liner:** Three command files enabling UI design contracts (ui-spec), 6-pillar scored audits (ui-review), and acceptance-criteria-driven test skeleton generation (add-tests).

## Tasks Completed

| # | Task | Commit | Files |
|---|------|--------|-------|
| 1 | Create ui-spec and ui-review commands | e012714 | commands/ui-spec.md, commands/ui-review.md |
| 2 | Create add-tests command | 9337531 | commands/add-tests.md |

## What Was Built

**ui-spec.md** (`/seraphim:ui-spec [phase]`)
Reads phase CONTEXT.md, RESEARCH.md, and PLAN.md files, then writes `UI-SPEC.md` to the phase directory. Output contains five sections: ASCII box wireframe, component table (Component/Props/State/Notes), interaction patterns (click/hover/submit/navigation), responsive breakpoints (mobile/tablet/desktop), and ARIA/keyboard/screen-reader accessibility notes.

**ui-review.md** (`/seraphim:ui-review [phase]`)
Scans SUMMARY.md and PLAN.md for `.tsx/.jsx/.css/.scss/.html` file references, guards against non-UI phases with an early exit message, reads each UI file, then writes `.planning/ui-reviews/{phase}-UI-REVIEW.md` containing a 6-pillar scored table (Layout, Typography, Color, Spacing, Accessibility, Responsiveness — each scored 1–5) with findings and recommendations. Prints overall score. Advisory, not blocking.

**add-tests.md** (`/seraphim:add-tests [phase]`)
Detects test framework from package.json (`vitest` > `jest` > framework-agnostic skeleton with install note), reads acceptance criteria from VERIFICATION.md and all PLAN.md `<acceptance_criteria>` / `<done>` blocks, checks for existing test files to avoid duplicating already-covered criteria, then writes test skeletons with Arrange/Act/Assert structure — one `it()` per criterion — at the correct test directory location for the detected framework.

## Deviations from Plan

None — plan executed exactly as written.

## Self-Check

- [x] `~/.claude/plugins/seraphim/commands/ui-spec.md` exists with `description:` frontmatter
- [x] `~/.claude/plugins/seraphim/commands/ui-review.md` exists with `description:` frontmatter
- [x] `~/.claude/plugins/seraphim/commands/add-tests.md` exists with `description:` and vitest/jest detection
- [x] Task 1 commit e012714 exists
- [x] Task 2 commit 9337531 exists

## Self-Check: PASSED
