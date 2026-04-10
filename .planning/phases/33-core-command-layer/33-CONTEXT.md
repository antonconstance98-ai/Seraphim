# Phase 33: Core Command Layer - Context

**Gathered:** 2026-04-09
**Status:** Ready for planning

<domain>
## Phase Boundary

Build the five core Seraphim commands (`/seraphim:seed`, `/seraphim:requirements`, `/seraphim:plan`, `/seraphim:execute`, `/seraphim:discuss`) plus supporting lib modules (`seed-store.js`, `requirements.js`, `wave-planner.js`). These commands enable users to capture ideas, define requirements, lock implementation decisions, generate wave-structured plans, and execute with parallel wave-based execution — all through native Seraphim commands.

</domain>

<decisions>
## Implementation Decisions

### Command Architecture
- **D-01:** Commands use subagent dispatch pattern — each command spawns a specialized agent, keeping command files lean (~50 lines frontmatter + prompt). Matches GSD pattern (discuss→planner, plan→planner+checker, execute→executor).
- **D-02:** Commands share a common `lib/` layer — new files `lib/seed-store.js`, `lib/requirements.js`, `lib/wave-planner.js` following the `roadmap.js` read/write atomic pattern.
- **D-03:** Commands discover project root via existing `config.js` pattern (reads `.seraphim/config.json`).
- **D-04:** All new commands use `.md` skill format with frontmatter, matching all 30 existing commands.

### Data Storage & Formats
- **D-05:** Seeds stored as `SEED-NNN.md` files in `.planning/seeds/` with YAML frontmatter (title, status, tags, created, trigger_conditions) + freeform body. `index.jsonl` for fast lookups. Per Phase 32 audit D-09.
- **D-06:** Requirements stored in `.seraphim/requirements.json` following `roadmap.json` atomic read/write pattern via `lib/requirements.js`. REQ-IDs as keys, categories as grouping. Per D-09.
- **D-07:** Wave dependencies extend `roadmap.json` feature objects with `waves: [{ id, tasks: [], depends_on: [] }]`. Kahn's algorithm in `lib/wave-planner.js` resolves execution order at plan time. Per D-09, PLAN-01, PLAN-02.
- **D-08:** `/seraphim:discuss` produces CONTEXT.md matching GSD discuss-phase output exactly — `<domain>`, `<decisions>`, `<code_context>`, `<specifics>`, `<deferred>` sections. Ensures compatibility with plan-phase consumers.

### Execution & Wave Parallelism
- **D-09:** `/seraphim:execute` matches GSD execute-phase pattern — discover plans, group by wave, spawn parallel agents per wave, sequential between waves. Reuses `gsd-executor` agent type.
- **D-10:** `/seraphim:plan` includes planner + checker verification loop with max 3 iterations (per PLAN-06). Checker is optional via config toggle.
- **D-11:** Seed trigger conditions stored as YAML frontmatter `trigger_conditions:` array with keyword/scope matchers. During `/seraphim:new-milestone`, scan all seeds and surface matches. Simple string matching.
- **D-12:** Commands are independent implementations that follow GSD patterns but call Seraphim lib files, not GSD tools. Avoids coupling to GSD internals.

### Claude's Discretion
- Internal agent prompt structures, error handling details, and UI output formatting are at Claude's discretion.

</decisions>

<code_context>
## Existing Code Insights

### Reusable Assets
- `lib/roadmap.js` — atomic read/write pattern (write to tmp, rename), readRoadmap/writeRoadmap — canonical reference for new lib files
- `lib/config.js` — project root discovery, config read/write
- `lib/decisions-logger.js` — buildRecord() for logging decisions to Neon
- `lib/push-client.js` — fire-and-forget Neon push
- `lib/phase-state.js` — phase state tracking

### Established Patterns
- `.md` command files with YAML frontmatter (name, description, allowed-tools)
- Subagent dispatch via prompt templates in command body
- Snake_case in JSON payloads, camelCase in JavaScript
- Atomic file writes via tmp + rename (roadmap.js pattern)

### Integration Points
- New commands go in `~/.claude/plugins/seraphim/commands/`
- New lib files go in `~/.claude/plugins/seraphim/lib/`
- `.seraphim/` directory for project-level data files
- `.planning/seeds/` for seed storage (directory exists from v3.0)

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

*Phase: 33-core-command-layer*
*Context gathered: 2026-04-09 via smart discuss*
