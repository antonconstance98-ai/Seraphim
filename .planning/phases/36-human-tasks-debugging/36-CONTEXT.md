# Phase 36: Human Tasks + Debugging - Context

**Gathered:** 2026-04-10
**Status:** Ready for planning

<domain>
## Phase Boundary

Enrich human task inbox with skills-to-learn, thought-prompt, and research-task fields. Build systematic debug and forensics commands with persistent state across session resets and auto-repair strategies.

</domain>

<decisions>
## Implementation Decisions

### Human Task Enrichment
- **D-01:** Extend existing HumanTask type in `push-client.js` payload with optional `skills_to_learn` (string[]), `thought_prompt` (string), `research_task` (string) fields. Neon `human_tasks` table gets matching TEXT columns via new migration.
- **D-02:** Enrichment fields populated at task creation time — when pipeline phases (Judge, Crucible) create human tasks, they analyze context and populate if relevant. Null if not applicable.
- **D-03:** Extend existing `/seraphim:inbox` command to display enrichment fields inline. Skills as tags, thought-prompt as expandable section.

### Debug & Forensics System
- **D-04:** Debug state persists in `.planning/debug/{slug}.md` with YAML frontmatter (status, hypothesis, findings, timeline). Each `/seraphim:debug` session reads existing state and appends new findings.
- **D-05:** Auto-repair uses strategy cascade — RETRY (re-run failed task), DECOMPOSE (split into subtasks), PRUNE (remove blocked dependency), ESCALATE (surface to human). Budget: max 2 retries, 1 decompose before escalation.
- **D-06:** `/seraphim:forensics` produces read-only root-cause report. Analyzes git history, error logs, state files. Writes to `.planning/debug/forensics/`. No code changes, no commits. Subagent with restricted tool access.
- **D-07:** Debug sessions can be spawned from UAT gap items — when verification finds gaps, it creates debug sessions linked to specific verification failures.

### Claude's Discretion
- Debug report formatting, auto-repair budget thresholds, and enrichment heuristics at Claude's discretion.

</decisions>

<code_context>
## Existing Code Insights

### Reusable Assets
- `lib/push-client.js` — Neon push payload, HumanTask type to extend
- `dashboard/lib/types.ts` — TypeScript types for Neon schema
- `dashboard/app/api/ingest/route.ts` — INSERT statements to update
- `commands/inbox.md` — existing inbox command to extend

### Established Patterns
- `.md` command files with YAML frontmatter
- Subagent dispatch for complex operations
- `.planning/debug/` for debug state (used by GSD debug system)

### Integration Points
- `dashboard/migrations/` for new Neon migration
- Pipeline phase commands (crucible.md, judge.md) for enrichment injection
- `.planning/debug/` for debug state persistence

</code_context>

<specifics>
## Specific Ideas

No specific requirements — user gave full Claude discretion.

</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope

</deferred>

---

*Phase: 36-human-tasks-debugging*
*Context gathered: 2026-04-10 via smart discuss*
