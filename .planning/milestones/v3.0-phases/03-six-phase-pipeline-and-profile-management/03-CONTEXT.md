# Phase 3: Six Phase Pipeline and Profile Management - Context

**Gathered:** 2026-04-04
**Status:** Ready for planning

<domain>
## Phase Boundary

Deliver six phase commands (`/seraphim:discover` through `/seraphim:crucible`), `/seraphim:run` with `--from` resume, structured output schemas for feedback loop parsing, all profile management commands, and non-code project type branching. First end-to-end pipeline run.

</domain>

<decisions>
## Implementation Decisions

### Non-Code Project Branching
- **D-01:** Forge for non-code projects generates prose/analysis to .md files in the project directory, one file per blueprint task. Commits like code — no difference in the git workflow.
- **D-02:** Crucible verification for non-code: completeness check — does output cover all blueprint sections? Adversarial: simulated critical peer review — poke holes in arguments, find unsupported claims, logical gaps.
- **D-03:** `project_type` field in blueprint.md drives branching. Values: `code`, `research`, `writing`, `science`, `mixed`. Forge and Crucible read this field to select behavior.

### Profile Switching
- **D-04:** Profile changes are allowed mid-pipeline run. The change takes effect on the NEXT phase — already-completed phases keep their original model assignments. No lock, no error.
- **D-05:** Phase-state.json records which profile was active when each phase executed, for audit and adaptive intelligence purposes.

### Terminal Output Style
- **D-06:** Phase completion output should be "in between" — not a minimal one-liner, not a full verbose dump. Show phase name, model used, key result (e.g., "3 approaches generated", "2 survived", "all tasks passed"), cost. Skip raw token counts and latency unless verbose mode.
- **D-07:** Headers and banners use the six-winged seraph concept. Seraphim has its own visual identity — not GSD banners. Design phase headers around the six wings theme (each phase = one wing).

### Claude's Discretion
- Command invocation pattern: Claude decides how phase commands call dispatch.js (Bash from .md, direct Node.js, etc.)
- Structured output format: Claude decides between JSON blocks, HTML comment markers, or YAML frontmatter — whatever enables reliable feedback loop parsing
- Banner/header exact ASCII art and formatting — Claude designs within the six-wing theme

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Design Spec
- `docs/specs/2026-04-04-seraphim-v3-design.md` — Full pipeline architecture, phase definitions, model roster, profile tables, dispatch flow, feedback loops

### Research
- `.planning/research/FEATURES.md` — Feature landscape with table stakes, differentiators, anti-features per category
- `.planning/research/ARCHITECTURE.md` — Component dependencies, build order, integration points
- `.planning/research/PITFALLS.md` — Plugin manifest placement, hook double-registration, structured output schema dependency

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `~/.claude/hooks/codex-exec.js` — Codex CLI wrapper pattern (fork source for codex-exec.js executor)
- `~/.claude/hooks/minimax-exec.js` — MiniMax API wrapper pattern (fork source for minimax-exec.js executor)
- `~/.claude/hooks/codex-review-gate.js` — Stop hook pattern showing how to evaluate model outputs and ALLOW/BLOCK

### Established Patterns
- Hook scripts use Node.js stdin/stdout JSON with 10s timeout guard
- All file writes use write-to-temp-then-renameSync (atomic on Linux)
- Token logging appends to JSONL with !== undefined field semantics

### Integration Points
- dispatch.js reads `.seraphim/config.json` (created in Phase 1)
- Phase commands consume executor output from Phase 2
- Structured output schemas must be parseable by Phase 4 feedback loops

</code_context>

<specifics>
## Specific Ideas

No specific requirements — open to standard approaches for command structure and output format.

</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope

</deferred>

---

*Phase: 03-six-phase-pipeline-and-profile-management*
*Context gathered: 2026-04-04*
