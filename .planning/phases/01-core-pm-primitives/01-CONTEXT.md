# Phase 1: Core PM Primitives - Context

**Gathered:** 2026-04-09
**Status:** Ready for planning

<domain>
## Phase Boundary

Deliver a terminal-first project management system: roadmap creation, feature queue, human task inbox, milestone archival, and pause/resume PM context. The PM layer observes pipeline execution but never gates or blocks it. After this phase, a developer can manage what to build and in what order from the terminal.

</domain>

<decisions>
## Implementation Decisions

### Feature Identification (D-01)
- **D-01:** Features use both auto-generated numeric IDs (feat-NNN) AND optional human-readable slugs. Schema: `{ id: "feat-001", slug: "auth-system", name: "Authentication System", ... }`. Commands accept either ID or slug for lookup.

### roadmap.json Schema (D-02)
- **D-02:** roadmap.json lives at `.seraphim/roadmap.json` per project. Structure: `{ milestones: [{ id, version, name, status, features: [{ id, slug, name, description, status, depends_on, priority }] }] }`. Status enum: planned | in-progress | complete | blocked.

### Feature Lifecycle (D-03)
- **D-03:** Auto-complete + notify: when all 6 pipeline phases pass (Crucible success), feature auto-marks as `complete` in roadmap.json AND writes a notification task to inbox ("Feature X completed -- review results"). No manual `/seraphim:done {feature}` required for pipeline completion.

### Add Feature UX (D-04)
- **D-04:** `/seraphim:add-feature` uses interactive prompts (step-by-step: name, milestone, description, priority). Follows the same pattern as `/seraphim:new-project`. Accept flags when provided, prompt for missing required fields.

### Inbox Layout (D-05)
- **D-05:** `/seraphim:inbox` groups by project first, then by task type within each project. Format: "Project A: 2 decisions, 1 research. Project B: 1 review."

### WIP Enforcement (D-06)
- **D-06:** WIP limit is a warning, not a block. `/seraphim:start` warns if limit exceeded but allows override. Default limit: 2. Configurable in `.seraphim/config.json`.

### Pause/Resume PM Context (D-07)
- **D-07:** `/seraphim:pause` extends state.json with `pm` block: `{ feature_id, milestone_version, progress: { phase, status } }`. `/seraphim:resume` reads this block to restore context. Must ship in Phase 1, not retrofitted later.

### PM as Read-Path (D-08)
- **D-08:** PM layer NEVER gates pipeline execution. Every PM command has implicit bypass -- pipeline commands work fine without roadmap.json existing. If roadmap.json is missing, PM commands create it on first use or return empty state.

### decisions-logger Extension (D-09)
- **D-09:** Add nullable `feature_id` field to `decisions-logger.js buildRecord()`. When a feature is active (in-progress), the logger includes its ID. When no feature is active, field is null. No breaking changes to existing records.

### Anti-Features (D-10)
- **D-10:** Enforced exclusions: no sprint/cycle system, no story points, no time-boxing, no drag-and-drop, no external PM sync, no Gantt charts, no time estimates.

### Claude's Discretion
- Terminal output formatting for roadmap tree and inbox display
- Exact interactive prompt flow for add-feature (which fields required vs optional)
- How milestone cost is computed and displayed in archival output

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Existing Plugin Code
- `~/.claude/plugins/seraphim/lib/phase-state.js` -- Pipeline phase tracking; PM reads this for feature progress
- `~/.claude/plugins/seraphim/lib/decisions-logger.js` -- Extend with feature_id field (D-09)
- `~/.claude/plugins/seraphim/lib/config.js` -- Config reader; PM uses for WIP limit
- `~/.claude/plugins/seraphim/commands/pause.md` -- Extend with PM context block (D-07)
- `~/.claude/plugins/seraphim/commands/resume.md` -- Read PM context block on resume
- `~/.claude/plugins/seraphim/commands/tasks.md` -- Existing human/AI task display; inbox extends this pattern
- `~/.claude/plugins/seraphim/commands/new-project.md` -- Interactive prompt pattern to follow for add-feature (D-04)

### Research
- `.planning/research/FEATURES.md` -- Feature landscape, MVP definition, anti-features
- `.planning/research/ARCHITECTURE.md` -- Integration approach, data model
- `.planning/research/PITFALLS.md` -- Second system effect prevention, ceremony creep
- `.planning/research/STACK.md` -- Zero new packages, file format patterns

### Requirements
- `.planning/REQUIREMENTS.md` -- ROAD-01 through ROAD-05, ROAD-07, QUEUE-01 through QUEUE-04, TASK-01 through TASK-04, ARCH-01 through ARCH-03, ARCH-06

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `phase-state.js`: Pipeline phase tracking -- PM reads completion status to derive feature progress
- `decisions-logger.js`: JSONL writer with `buildRecord()` -- extend with feature_id, don't replace
- `config.js`: Reads `.seraphim/config.json` -- add `max_wip` field
- `tasks.md`: Reads SERAPHIM:HUMAN_TASKS markers from forge-log.md -- inbox aggregates across features
- `markers.js`: Task marker parsing utilities

### Established Patterns
- All state files in `.seraphim/` per project (config.json, state.json, decisions.jsonl)
- Slash commands are Markdown files in `commands/` directory
- Interactive prompts follow the pattern in `new-project.md`
- Synchronous file writes for crash safety (phase-state.js pattern)

### Integration Points
- `run.md` invokes pipeline phases -- needs to read active feature from roadmap.json for context
- `pause.md` writes state.json -- extend with PM block
- `resume.md` reads state.json -- read PM block
- `decisions-logger.js` `buildRecord()` -- add feature_id parameter

</code_context>

<specifics>
## Specific Ideas

- Feature IDs: both numeric (feat-001) and slug ("auth-system") -- user chose "both"
- Inbox sorted by project then type -- matches "what do I need to do per project" mental model
- Auto-complete + notify pattern: pipeline success auto-completes but user still sees it in inbox

</specifics>

<deferred>
## Deferred Ideas

None -- discussion stayed within phase scope

</deferred>

---

*Phase: 01-core-pm-primitives*
*Context gathered: 2026-04-09*
