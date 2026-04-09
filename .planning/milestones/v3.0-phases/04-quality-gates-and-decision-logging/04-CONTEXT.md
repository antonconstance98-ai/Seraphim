# Phase 4: Quality Gates and Decision Logging - Context

**Gathered:** 2026-04-04
**Status:** Ready for planning

<domain>
## Phase Boundary

Deliver between-task checkpoints in Forge (runtime + static review), retry-with-feedback, Judge->Envision and Crucible->Forge feedback loops with hard caps and persisted counters, human escalation on cap exceeded, and decisions.jsonl logging with quality signals.

</domain>

<decisions>
## Implementation Decisions

### Claude's Discretion
- **D-01:** Checkpoint scope (what constitutes a "task" for checkpoint purposes) — Claude decides based on blueprint task granularity
- **D-02:** Feedback context format (how much prior-phase output gets injected into retry prompts) — Claude decides based on token budget and model context limits
- **D-03:** Cost-gate before loops — Claude decides whether to warn before expensive retry iterations or just run and log
- **D-04:** decisions.jsonl granularity — Claude decides whether to log per-phase-completion, per-executor-call, or both. Must include all required fields: phase, model, profile, tokens_in, tokens_out, cost_usd, latency_ms, outcome, retry_count, loop_count, quality signals
- **D-05:** Data integrity validator implementation approach — Claude decides detection method and reporting format

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Design Spec
- `docs/specs/2026-04-04-seraphim-v3-design.md` — Feedback loop definitions, checkpoint flow, hard cap rules

### Research
- `.planning/research/FEATURES.md` §Quality Gates and §Feedback Loops — table stakes, hard cap rationale (industry standard max 2-3)
- `.planning/research/PITFALLS.md` — Feedback loop counter persistence (must be on disk, not in-memory), cost formula pitfalls across 9 models
- `.planning/research/SUMMARY.md` §Critical Pitfalls — loop counter lost across sessions, token cost shared formula producing negative costs

### Prior Phase Context
- `.planning/phases/03-six-phase-pipeline-and-profile-management/03-CONTEXT.md` — Structured output format decision (affects how feedback loops parse phase outputs)

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `~/.claude/hooks/codex-review-gate.js` — Existing checkpoint pattern with ALLOW/BLOCK protocol and task-type-aware depth
- `~/.claude/hooks/codex-wave-validator.js` — Wave validation pattern for checking task completion
- `~/.claude/hooks/decision-logger.js` — Existing decision logging to JSONL (fork/adapt for decisions.jsonl)
- `~/.claude/hooks/minimax-post-scan.js` — Static code review via MiniMax (pattern for checkpoint static review)

### Established Patterns
- phase-state.js (from Phase 1) persists loop counters to `.seraphim/phases/{N}/state.json`
- JSONL append-only with writeHookSignal pattern (last-wins merge on read)
- Fail-closed for execution tasks, fail-open for review tasks

### Integration Points
- checkpoint.js calls dispatch.js to resolve checkpoint models per profile
- Feedback loops read structured output schemas from Phase 3
- decisions.jsonl feeds Phase 6 adaptive intelligence

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

*Phase: 04-quality-gates-and-decision-logging*
*Context gathered: 2026-04-04*
