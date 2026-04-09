---
phase: 01-plugin-scaffold-and-infrastructure
plan: 01
subsystem: plugin-core
tags: [scaffold, manifest, config, models, profiles]
requires: []
provides: [plugin-manifest, models-config, profiles-config, plugin-scaffold]
affects: [all-subsequent-plans]
tech_stack:
  added: []
  patterns: [claude-plugin-manifest, json-config-files]
key_files:
  created:
    - ~/.claude/plugins/seraphim/.claude-plugin/plugin.json
    - ~/.claude/plugins/seraphim/config/models.json
    - ~/.claude/plugins/seraphim/config/profiles.json
  modified: []
decisions:
  - "Manifest placed at .claude-plugin/plugin.json (not root) per Pitfall 1 research"
  - "No hooks key in plugin.json to avoid double-registration silent failure (Pitfall 2)"
  - "All 9 model IDs cross-referenced in profiles.json — no dangling references"
metrics:
  duration_min: 5
  completed_date: "2026-04-05"
  tasks_completed: 3
  tasks_total: 3
  files_created: 3
  files_modified: 0
---

# Phase 01 Plan 01: Plugin Scaffold and Infrastructure Summary

**One-liner:** Plugin directory scaffold at ~/.claude/plugins/seraphim/ with .claude-plugin/plugin.json manifest, nine-model roster in models.json, and five preset profiles plus naked template in profiles.json.

## What Was Built

The foundational directory structure and static configuration files for the Seraphim plugin. All subsequent plans depend on these files being correctly placed.

### Plugin Manifest

`~/.claude/plugins/seraphim/.claude-plugin/plugin.json` — the Claude Code plugin registration file. Placed in the `.claude-plugin/` subdirectory as required by the plugin system (verified against 14 installed plugins on this machine). Contains `name`, `version`, `description`, and `author` only — no `hooks` key to avoid the double-registration silent failure.

### Directory Scaffold

Seven subdirectories created under the plugin root: `config/`, `commands/`, `executors/`, `hooks/`, `lib/`, `agents/`, `tools/`. These are empty scaffolds for subsequent plans to populate.

### models.json — Nine-Model Roster

`~/.claude/plugins/seraphim/config/models.json` defines all nine models with their mechanism, pricingKey, isOpus flag, capabilities array, and cost_per_mtok. Key metadata enforced:
- Only `claude-opus-4-6` has `isOpus: true`
- `minimax-m2.7` has `temperature: 0.01` (API rejects exactly 0)
- `codex-gpt-5.4` timeout 300s, `qwen-3.5-27b` timeout 180s, `minimax-m2.7` timeout 90s
- `qwen-3.5-27b` has `requiresGPU: true`

### profiles.json — Five Presets and Naked Template

`~/.claude/plugins/seraphim/config/profiles.json` defines six profiles: performance, balanced, moderate, budget, frugal, and naked. Each profile has exactly ten sub-phase slots and an `opusFallback` field. The naked profile has all slots set to `null`. All model IDs used in profiles cross-referenced against models.json — all valid.

## Tasks Completed

| Task | Name | Commit | Files |
|------|------|--------|-------|
| 1 | Create plugin directory scaffold and manifest | 65d1eba | .claude-plugin/plugin.json + 7 dirs |
| 2 | Create models.json with nine-model roster | 02bde5a | config/models.json |
| 3 | Create profiles.json with five presets and naked template | b9706bf | config/profiles.json |

## Deviations from Plan

None — plan executed exactly as written.

Note: The plugin files live at `~/.claude/plugins/seraphim/` outside the project repo. A git repo was initialized at the plugin root to satisfy the per-task commit protocol. The project repo at `~/projects/Seraphim/` tracks planning artifacts; the plugin repo tracks plugin source files.

## Known Stubs

None. All three files are complete static configuration with no placeholder values that flow to rendering. The scaffold directories are intentionally empty — they are populated by subsequent plans.

## Self-Check: PASSED

| Item | Status |
|------|--------|
| ~/.claude/plugins/seraphim/.claude-plugin/plugin.json | FOUND |
| ~/.claude/plugins/seraphim/config/models.json | FOUND |
| ~/.claude/plugins/seraphim/config/profiles.json | FOUND |
| commit 65d1eba (scaffold + manifest) | FOUND |
| commit 02bde5a (models.json) | FOUND |
| commit b9706bf (profiles.json) | FOUND |
