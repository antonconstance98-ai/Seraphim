# Phase 35: Phase Management + Config + UI Tooling - Context

**Gathered:** 2026-04-10
**Status:** Ready for planning

<domain>
## Phase Boundary

Build phase lifecycle commands (add/insert/remove-phase, complete-milestone, pr-branch, health, workstreams, manager), configuration commands (settings, set-profile), and UI/quality tooling commands (ui-spec, ui-review, add-tests). All commands follow the Seraphim `.md` skill format and call Seraphim lib files.

</domain>

<decisions>
## Implementation Decisions

### Phase Lifecycle Management
- **D-01:** add/insert/remove-phase directly manipulate ROADMAP.md — parse markdown, insert/remove phase section, renumber subsequent phases, update dependency references. No intermediate JSON.
- **D-02:** `/seraphim:complete-milestone` archives to `.planning/milestones/vX.Y-ROADMAP.md` + git tag + cost attribution from session JSONL. Cleans up phase directories. Matches GSD complete-milestone pattern.
- **D-03:** `/seraphim:pr-branch` cherry-picks non-.planning/ commits to a clean branch. User gets a PR-ready branch without planning artifacts.
- **D-04:** Workstreams use `.planning/workstreams/` directory with state files per workstream, each tracking independent phase progress.

### Configuration System
- **D-05:** Model profiles stored as presets in config.json — quality/balanced/budget/inherit profiles map to model selections per command type (planner, executor, reviewer). `/seraphim:settings` writes profile choice.
- **D-06:** Workflow settings as toggle map — `skip_discuss`, `auto_advance`, `ui_phase`, `ui_review`, `plan_checker`, `research_enabled`, `parallelization`. All read via config-get, written via `/seraphim:settings`.
- **D-07:** `/seraphim:health` validates `.planning/` structure — checks for ROADMAP.md, STATE.md, REQUIREMENTS.md, phase directories, orphaned plans. Reports issues with repair suggestions.

### UI & Quality Tooling
- **D-08:** `/seraphim:ui-spec` produces UI-SPEC.md with layout wireframes (ASCII), component list, interaction patterns, responsive breakpoints, accessibility notes. Consumed by executors as design contract.
- **D-09:** `/seraphim:ui-review` runs 6-pillar audit — layout, typography, color, spacing, accessibility, responsiveness. Produces scored UI-REVIEW.md. Advisory, not blocking.
- **D-10:** `/seraphim:add-tests` generates test files based on VERIFICATION.md acceptance criteria and PLAN.md done-criteria. Auto-detects test framework (jest/vitest).

### Claude's Discretion
- Internal prompt structures, error messages, manager UI layout, and display formatting at Claude's discretion.

</decisions>

<code_context>
## Existing Code Insights

### Reusable Assets
- `lib/roadmap.js` — readRoadmap/writeRoadmap for milestone/feature manipulation
- `lib/config.js` — config read/write, project root discovery
- `lib/requirements.js` — REQ-ID CRUD (Phase 33)
- `lib/wave-planner.js` — dependency resolution (Phase 33)
- Existing `close-milestone.md`, `roadmap.md` commands — patterns to follow/replace

### Established Patterns
- `.md` command files with YAML frontmatter
- `node -e` inline scripts for lib calls
- Subagent dispatch for complex operations
- `.planning/` for planning artifacts, `.seraphim/` for project data

### Integration Points
- Commands in `~/.claude/plugins/seraphim/commands/`
- `.planning/config.json` for workflow settings
- `.planning/milestones/` for archived milestones
- `.planning/workstreams/` for workstream state

</code_context>

<specifics>
## Specific Ideas

No specific requirements — user gave full Claude discretion on all areas.

</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope

</deferred>

---

*Phase: 35-phase-management-config-ui-tooling*
*Context gathered: 2026-04-10 via smart discuss*
