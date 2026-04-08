---
phase: 03-six-phase-pipeline-and-profile-management
plan: 05
subsystem: config
tags: [profiles, commands, cost-management]

requires:
  - phase: 01-plugin-scaffold-and-infrastructure
    provides: config.js read/write, profiles.json, models.json
provides:
  - set-profile slash command for switching cost profiles
  - show-profile slash command for viewing assignments and costs
  - override slash command for per-phase model overrides
affects: [run, forge, crucible, discover, envision, judge, architect]

tech-stack:
  added: []
  patterns: [slash-command-md-pattern]

key-files:
  created:
    - ~/.claude/plugins/seraphim/commands/set-profile.md
    - ~/.claude/plugins/seraphim/commands/show-profile.md
    - ~/.claude/plugins/seraphim/commands/override.md
  modified: []

key-decisions:
  - "Used actual profiles.json phase slot names (discover, envision, etc.) not the 10-slot names from earlier research"
  - "Qwen availability check uses curl to ollama API rather than Node.js executor"
  - "Cost estimates assume ~2K tokens in + 2K tokens out per phase as baseline"

patterns-established:
  - "Command files: YAML frontmatter with description, argument-hint, allowed-tools + markdown instructions"

requirements-completed: [PROF-01, PROF-02, PROF-03, PROF-04, PROF-05, PROF-08]

duration: 8min
completed: 2026-04-08
---

# Plan 05: Profile Management Commands

**Three slash commands (set-profile, show-profile, override) give users full control over which models run each pipeline phase, with cost estimates, Qwen availability warnings, and per-phase override support.**

## What Was Built

- **set-profile.md**: Lists built-in + custom profiles, validates selection, warns on Qwen unavailability for budget/frugal profiles, updates config, records profile change events, prints phase-to-model assignment table with opus fallback handling
- **show-profile.md**: Displays current profile with all phase assignments, cost estimates from models.json, override markers, and Qwen availability status
- **override.md**: Sets per-phase model overrides with validation of both phase-slot and model-id, supports `--clear` to remove overrides, shows updated assignment table

## Self-Check: PASSED

All acceptance criteria verified:
- All three files exist with correct frontmatter
- set-profile references profiles.json, custom profiles, opus_enabled, qwen availability
- show-profile includes cost estimates and override display
- override validates phase-slot and model-id, supports --clear
