# Phase 10: Context Management and Token Optimization - Context

**Gathered:** 2026-04-08
**Status:** Ready for planning

<domain>
## Phase Boundary

Research and implement strategies to minimize token usage across the nine-model pipeline without degrading output quality. Four strategy tracks: inter-phase summarization, prompt template optimization, smart caching, and model-specific optimization. Tiered context flow between phases. Multiple measurement strategies for accuracy.

</domain>

<decisions>
## Implementation Decisions

### Token Reduction Strategies
- **D-01:** All four strategies are in scope: (1) inter-phase summarization, (2) prompt template optimization, (3) smart caching, (4) model-specific optimization.
- **D-02:** Inter-phase summarization: generate structured summaries between phases instead of passing full outputs. Reduces tokens for downstream phases that don't need full detail.
- **D-03:** Prompt template optimization: audit and compress system prompts and phase instructions. Remove redundancy, use concise instructions, leverage model-specific prompt efficiency patterns.
- **D-04:** Smart caching: cache reusable context (project config, model roster, common instructions) so they're not re-sent every phase. Leverage Anthropic's prompt caching for Claude-hosted phases.
- **D-05:** Model-specific optimization: tailor input size to each model's sweet spot. Don't send 32K tokens to a model that works best at 8K. Each model has different context windows and pricing.

### Cross-Phase Context Flow
- **D-06:** Tiered by phase transition. Some transitions need full context (Judge needs full Envision output to evaluate approaches). Others can use summaries (Forge just needs the blueprint, not the full Judge rationale). Configure per-transition.
- **D-07:** Research phase determines the optimal tier for each of the five transitions (Discover→Envision, Envision→Judge, Judge→Architect, Architect→Forge, Forge→Crucible) based on actual token usage and quality impact analysis.

### Measurement Approach
- **D-08:** Claude's discretion on methodology, but use as many tracking strategies as possible for maximum accuracy. Combine historical baselines (decisions.jsonl), before/after comparisons, and target-based tracking.
- **D-09:** Track token reduction per strategy (so you can see which optimization contributed most), not just aggregate savings.
- **D-10:** Results visible on the Phase 7 dashboard — token usage trends and cost savings panel.

### Claude's Discretion
- Specific measurement methodology (statistical approach, comparison design)
- Priority ordering of the four strategies (which to implement first)
- Per-transition context tier assignments (full vs summary vs hybrid)
- Prompt compression techniques (model-specific tricks, instruction reformatting)
- Caching implementation (Anthropic prompt caching, local file caching, or both)

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Design Spec
- `docs/specs/2026-04-04-seraphim-v3-design.md` — Phase definitions, model roster, profile tables (token budgets implied by pricing)

### Prior Context
- `.planning/phases/03-six-phase-pipeline-and-profile-management/03-CONTEXT.md` — Pipeline structure, phase output schemas, structured markers for feedback loop parsing
- `.planning/phases/06-adaptive-intelligence/06-CONTEXT.md` — Dashboard integration, analysis frequency, decisions.jsonl schema

### Existing Infrastructure
- v2.0 context compression via MiniMax (Phase 12 of v2.0) — pattern reference for compression approach
- `.planning/milestones/v2.0-phases/12-context-compression/` — Prior compression implementation details

### Cost Tracking
- `.planning/phases/04-quality-gates-and-decision-logging/04-CONTEXT.md` — decisions.jsonl schema (the measurement baseline source)

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- v2.0 MiniMax context compression pattern (hooks-based) — can inform summarization approach
- token-logger.js handles four incompatible token count schemas across providers
- pricing.js has per-provider cost functions for all nine models

### Established Patterns
- JSONL append-only logging with session correlation
- Per-provider token counting (Anthropic, Gemini, OpenAI/MiniMax, ollama schemas)
- Phase-state.js tracks per-phase execution metadata

### Integration Points
- decisions.jsonl provides historical baseline data
- token-log.jsonl provides per-call token counts
- Pipeline orchestrator (Phase 3, plan 03-06) controls context passing between phases
- Dashboard (Phase 7) displays optimization results

</code_context>

<specifics>
## Specific Ideas

No specific requirements — open to whatever research finds as the most effective optimization approaches. User wants maximum accuracy in measurement, not a simplified metric.

</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope

</deferred>

---

*Phase: 10-context-management-and-token-optimization*
*Context gathered: 2026-04-08*
