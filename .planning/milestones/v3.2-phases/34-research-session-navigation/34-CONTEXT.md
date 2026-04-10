# Phase 34: Research + Session + Navigation - Context

**Gathered:** 2026-04-09
**Status:** Ready for planning

<domain>
## Phase Boundary

Build research commands (scope + run with interrogation gate), session management (pause/resume/report), navigation routing (next/do/progress), and codebase mapping. All commands follow the Seraphim `.md` skill format, call Seraphim lib files, and produce outputs consumable by existing pipeline commands.

</domain>

<decisions>
## Implementation Decisions

### Research System
- **D-01:** Two-command gate enforced via state file — `/seraphim:research-scope` writes `.seraphim/research.json` entry with `status: "scoped"`. `/seraphim:research-run` checks for scoped status before executing. Human interrogation cannot be skipped.
- **D-02:** research-scope produces structured scope doc in research.json — topic, questions, constraints, categories. Human approves scope before AI research runs.
- **D-03:** `/seraphim:map-codebase` spawns 3-4 parallel mapper agents, each analyzing a focus area (structure, conventions, stack, concerns). Results written to `.planning/codebase/`. Matches GSD codebase-mapper pattern.
- **D-04:** `lib/research-tracker.js` stores state in `.seraphim/research.json` following requirements.js atomic read/write pattern. Each item: id, topic, status, scope, results.

### Session Management
- **D-05:** `/seraphim:pause` captures HANDOFF.json (phase, plan, task position, uncommitted changes, active branch, open decisions) + `.continue-here.md` (human-readable resumption guide).
- **D-06:** `/seraphim:resume` reads HANDOFF.json, injects into Claude context, displays summary, suggests next action. Deletes handoff files after successful resume.
- **D-07:** `/seraphim:session-report` reads git log since session start, counts commits, lists files changed, estimates token spend from session JSONL. Writes to `.planning/session-reports/`.

### Navigation & Routing
- **D-08:** `/seraphim:next` uses state-machine logic — reads STATE.md + ROADMAP.md, checks for missing artifacts (CONTEXT.md/PLAN.md/SUMMARY.md/VERIFICATION.md), routes to earliest missing artifact's command.
- **D-09:** `/seraphim:do` uses keyword matching + intent classification — scans input for action verbs (plan, execute, debug, seed, research), routes to matching command. Fallback: present top 3 matches for user selection.
- **D-10:** `/seraphim:progress` shows phase table + completion bars + next action — reads ROADMAP.md phases, calculates completion %, shows current position, suggests next command.

### Claude's Discretion
- Internal prompt structures, error messages, and display formatting at Claude's discretion.

</decisions>

<code_context>
## Existing Code Insights

### Reusable Assets
- `lib/roadmap.js` — atomic read/write pattern, readRoadmap/writeRoadmap
- `lib/requirements.js` — atomic requirements.json CRUD (built in Phase 33)
- `lib/seed-store.js` — SEED file management (built in Phase 33)
- `lib/config.js` — project root discovery
- Existing `pause.md` and `resume.md` commands — may need updating or replacing

### Established Patterns
- `.md` command files with YAML frontmatter
- `node -e` inline scripts for lib calls within commands
- Subagent dispatch for complex operations
- `.seraphim/` for project-level data, `.planning/` for planning artifacts

### Integration Points
- Commands in `~/.claude/plugins/seraphim/commands/`
- New lib file: `lib/research-tracker.js`
- `.seraphim/research.json` for research state
- `.planning/codebase/` for mapper output
- `.planning/session-reports/` for session reports

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

*Phase: 34-research-session-navigation*
*Context gathered: 2026-04-09 via smart discuss*
