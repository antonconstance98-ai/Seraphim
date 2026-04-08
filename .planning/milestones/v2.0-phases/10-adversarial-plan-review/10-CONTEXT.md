# Phase 10: Adversarial Plan Review - Context

**Gathered:** 2026-04-03
**Status:** Ready for planning

<domain>
## Phase Boundary

Modify the multi-round plan review system so Round 2 (adversarial/devil's advocate) is handled by MiniMax instead of Codex. Round 1 (constructive) stays Codex. MiniMax's self-evolution training gives it genuinely different reasoning patterns, making it a better adversary than Codex reviewing its own output twice.

</domain>

<decisions>
## Implementation Decisions

### Model-per-round configuration
- **D-01:** Modify `codex-multi-round-reviewer.js` to accept a `model` parameter per round. Round 1: Codex (constructive). Round 2: MiniMax (adversarial). The round config becomes `[{ model: 'codex', type: 'constructive' }, { model: 'minimax', type: 'adversarial' }]`.
- **D-02:** Applies to BOTH `codex-plan-reviewer.js` (GSD) and `codex-superpowers-plan-reviewer.js` (Superpowers). Same multi-round reviewer module, same config.

### Think tag handling
- **D-03:** Preserve everything. MiniMax's full `<think>` reasoning chain appears in REVIEWS.md output. User sees MiniMax's complete thought process alongside its findings. Full transparency.
- **D-04:** Use `reasoning_split: true` in the MiniMax API call (via `extra_body`) to get structured reasoning in the `reasoning_details` field. Preserve BOTH the structured field AND any inline `<think>` tags in conversation history.
- **D-05:** When passing Round 1 Codex findings to MiniMax for Round 2, include the full findings text. MiniMax receives: "Here are the constructive review findings from Round 1: [Codex output]. Now conduct an adversarial review — poke holes, find flaws, challenge assumptions, act as devil's advocate."

### Review state tracking
- **D-06:** `review-state.json` updated to track which model ran each round: `rounds[].model` field added (`'gpt-5.4'` or `'minimax-m2.7'`).
- **D-07:** Token logging per round tracks the correct model. Round 1 logged as `model: 'gpt-5.4'`, Round 2 as `model: 'minimax-m2.7'`.

### Fallback behavior
- **D-08:** If MiniMax is unavailable for Round 2, fall back to Codex adversarial (current behavior). The plan review degrades to same-model-twice rather than failing entirely. Log the fallback event.

### Claude's Discretion
- Adversarial prompt wording (how aggressively MiniMax should challenge)
- Whether to add a Round 3 synthesis option in the future (deferred)
- Exact format of `<think>` content in REVIEWS.md (inline vs collapsible section)

</decisions>

<specifics>
## Specific Ideas

- MiniMax's adversarial review should feel like a genuine "red team" — not just finding issues but actively trying to break the plan, surface hidden assumptions, and identify production failure scenarios
- The `<think>` reasoning chain is valuable because it shows HOW MiniMax arrived at its challenges, not just WHAT it found. This transparency is the point.
- Round 1 findings should be presented to MiniMax as context, not as constraints. MiniMax should feel free to disagree with Codex's Round 1 assessment.

</specifics>

<canonical_refs>
## Canonical References

### Phase 8 Foundation (dependency)
- `.planning/phases/08-minimax-foundation/08-CONTEXT.md` — `runMinimax()` interface, `reasoning_split` support

### Existing Review Infrastructure
- `~/.claude/hooks/codex-multi-round-reviewer.js` — Shared review orchestrator to modify (currently hardcoded to Codex for both rounds)
- `~/.claude/hooks/codex-plan-reviewer.js` — GSD SubagentStop hook that calls multi-round-reviewer
- `~/.claude/hooks/codex-superpowers-plan-reviewer.js` — Superpowers SubagentStop hook that calls multi-round-reviewer

### Research
- `minimax-m2.7-synthesis.md` §5 — `<think>` tag fragility warning, `reasoning_split` usage
- `research/deep-research-report(3).md` — Multi-model routing patterns, MiniMax prompt templates for reviews

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `codex-multi-round-reviewer.js` v3.0 — Already has round loop, state management via `review-state.json`, token logging per round. Needs model parameter added to round config.

### Established Patterns
- Review state persisted as JSON: `{ schema_version, phase, max_rounds, current_round, rounds[] }`
- Each round record: `{ round, review_type, completed_at, issue_count, has_high_issues, text }`
- Early exit on clean Round 1 (0 issues → skip Round 2)

### Integration Points
- `codex-multi-round-reviewer.js` currently calls `runCodexExec()` for every round. Round 2 call changes to `runMinimax()` from `minimax-exec.js`.
- `review-state.json` in `.planning/` directory — add `model` field to round records.

</code_context>

<deferred>
## Deferred Ideas

- Round 3 synthesis (merge R1+R2 into unified assessment) — evaluate after v2.0 ships
- Using MiniMax for BOTH rounds when Codex is rate-limited (already handled by D-08 fallback)

</deferred>

---

*Phase: 10-adversarial-plan-review*
*Context gathered: 2026-04-03*
