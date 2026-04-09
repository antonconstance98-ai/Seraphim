# Phase 1: Plugin Scaffold and Infrastructure - Context

**Gathered:** 2026-04-04
**Status:** Ready for planning

<domain>
## Phase Boundary

Deliver the plugin manifest, dispatch.js router, profiles.json with five presets + user-created named profiles, config.js, phase-state.js, models.json, and the `/seraphim:new-project` guided setup command. The plugin must load in Claude Code with `/seraphim:` namespace.

</domain>

<decisions>
## Implementation Decisions

### Init Command UX
- **D-01:** `/seraphim:new-project` uses a guided setup flow with 4 questions: profile selection, project type (code/research/writing/science/mixed), opus_enabled toggle, and max_loops preference. Creates `.seraphim/config.json` with full config.

### Custom Profiles
- **D-02:** Users can create multiple named custom profiles with completely different model wiring. Not limited to one "custom" profile — users can create, name, and save as many profiles as they want.
- **D-03:** The five built-in profiles (Performance, Balanced, Moderate, Budget, Frugal) are presets that ship with the plugin. Users can also create a "naked" profile where every slot is empty and the user fills in which model goes where.
- **D-04:** Custom profiles are stored per-project in `.seraphim/profiles/` as individual JSON files (e.g., `my-research-profile.json`). Built-in profiles live in the plugin at `config/profiles.json`.
- **D-05:** `/seraphim:set-profile` lists both built-in and user-created profiles for selection.

### Claude's Discretion
- Plugin directory layout (research found `.claude-plugin/plugin.json` for manifest — Claude follows research findings)
- dispatch.js internal architecture (resolution order: override > opus_enabled > profile is locked from roadmap)
- phase-state.js persistence format
- models.json schema design

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Design Spec
- `docs/specs/2026-04-04-seraphim-v3-design.md` — Plugin structure, dispatch flow, config format, profile tables, model roster

### Research
- `.planning/research/STACK.md` — Plugin manifest schema, dispatch.js lazy-require pattern
- `.planning/research/ARCHITECTURE.md` — Component map, build order, plugin.json placement
- `.planning/research/PITFALLS.md` — Plugin manifest in wrong directory (silent failure), hook double-registration

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- No existing plugin structure to reference — this is the first Claude Code plugin in this project
- `~/.claude/hooks/codex-exec.js` — Pattern for how hooks read settings and resolve paths (fork reference)

### Established Patterns
- `.claude/settings.json` stores project-scope config (codex, minimax blocks are siblings)
- JSON config files with validation and defaults

### Integration Points
- Plugin must register in Claude Code's plugin system
- `.seraphim/config.json` consumed by dispatch.js (Phase 2+)
- phase-state.js consumed by feedback loops (Phase 4)

</code_context>

<specifics>
## Specific Ideas

- User wants a "naked" empty profile option where they fill in every model slot themselves
- User wants named custom profiles they can create and switch between

</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope

</deferred>

---

*Phase: 01-plugin-scaffold-and-infrastructure*
*Context gathered: 2026-04-04*
