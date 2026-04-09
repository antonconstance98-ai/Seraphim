# Phase 5: Session Commands and Hook Consolidation - Context

**Gathered:** 2026-04-04
**Status:** Ready for planning

<domain>
## Phase Boundary

Deliver all session management commands (`/seraphim:help`, `/seraphim:history`, `/seraphim:pause`, `/seraphim:resume`, `/seraphim:status`), and atomically retire seven redundant v2.0 hooks while registering plugin equivalents.

</domain>

<decisions>
## Implementation Decisions

### Claude's Discretion
- **D-01:** All implementation decisions for this phase deferred to Claude. Session commands are straightforward utility commands. Hook retirement sequence is well-defined in research (atomic swap in settings.json, archive copies preserved).
- **D-02:** `/seraphim:help` format, `/seraphim:history` display format (table vs timeline), history depth, and data scope (per-project vs global) — Claude decides based on what's most useful.
- **D-03:** Hook retirement trigger criteria — Claude decides (e.g., after Phase 4 quality gates are proven, or after N successful pipeline runs).
- **D-04:** `/seraphim:pause` and `/seraphim:resume` serialization format — Claude decides, must be compatible with phase-state.js from Phase 1.

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Design Spec
- `docs/specs/2026-04-04-seraphim-v3-design.md` §Hooks Consolidation — which hooks retire, which are kept/forked

### Research
- `.planning/research/ARCHITECTURE.md` — Seven hooks retire atomically, never both active simultaneously
- `.planning/research/PITFALLS.md` — Hook double-registration silent failure, old hooks firing alongside new pipeline

### Existing Hooks (Retirement Targets)
- `~/.claude/hooks/codex-review-gate.js` — Replaced by Crucible phase
- `~/.claude/hooks/codex-plan-reviewer.js` — Replaced by Judge phase
- `~/.claude/hooks/codex-multi-round-reviewer.js` — Replaced by Judge phase
- `~/.claude/hooks/minimax-post-scan.js` — Replaced by Crucible adversarial pass
- `~/.claude/hooks/minimax-compress.js` — Each executor manages own context
- `~/.claude/hooks/codex-router.js` — Replaced by dispatch.js
- `~/.claude/hooks/codex-wave-validator.js` — Replaced by Forge checkpoint logic

### Prior Phase Context
- `.planning/phases/01-plugin-scaffold-and-infrastructure/01-CONTEXT.md` — phase-state.js persistence format (pause/resume must be compatible)

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `~/.claude/hooks/codex-cost-reporter.js` — Session cost reporting pattern (reference for /seraphim:history)
- `~/.claude/settings.json` — Hook registration format (reference for atomic retirement)

### Established Patterns
- Settings.json hook groups with timeout values
- Session reports written as markdown
- Atomic config edits (read → modify → write-temp → rename)

### Integration Points
- `/seraphim:status` reads config.js, phase-state.js, and executor availability
- `/seraphim:history` reads decisions.jsonl and token-log.jsonl
- Hook retirement edits `~/.claude/settings.json` directly
- Archive copies go to a known rollback path

</code_context>

<specifics>
## Specific Ideas

No specific requirements — open to standard approaches.

</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope

</deferred>

---

*Phase: 05-session-commands-and-hook-consolidation*
*Context gathered: 2026-04-04*
