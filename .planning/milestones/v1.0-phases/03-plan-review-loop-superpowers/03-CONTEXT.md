# Phase 3: Plan Review Loop & Superpowers - Context

**Gathered:** 2026-04-02
**Status:** Ready for planning

<domain>
## Phase Boundary

Upgrade the Phase 2 single-pass Codex review into a multi-round Opus-Codex plan review loop, and integrate Codex into Superpowers parallel agent dispatch. This phase delivers: 2-round review loop (constructive + adversarial) with early exit, typed handoff specs with `decisions_not_taken`, review state persistence, Superpowers parallel agent routing to GPT-5.4-mini, and Superpowers plan review using the same loop. No cost reporting (Phase 4).

</domain>

<decisions>
## Implementation Decisions

### Review Loop Dynamics
- **D-01:** Loop follows a **5-step flow**: Opus drafts → Codex critiques (constructive) → Opus revises → Codex adversarial review (poke holes, edge cases) → Opus final revision. Two distinct Codex review types — constructive then adversarial
- **D-02:** **Early exit when clean** — if Codex's first constructive review finds zero issues, the adversarial round is skipped. Saves time and tokens on clean plans
- **D-03:** **Opus always wins** after the last round. Codex's unresolved concerns go into the `decisions_not_taken` section of the handoff spec. Matches the "Opus is architect" rule from Phase 1

### Superpowers Routing
- **D-04:** Three Superpowers parallel agent types route to **GPT-5.4-mini via API**: hypothesis testing, code review threads, and verification checks. All three are good fits for a fast, cheap model
- **D-05:** **Escalate to Opus on low confidence** — if GPT-5.4-mini gives low-confidence or unclear results, re-run on Opus. Only triggers when needed, ensuring quality while optimizing cost

### Loop Visibility
- **D-06:** User sees **milestone updates** during the loop: "Round 1: Codex reviewing..." then "Round 1: 3 suggestions. Opus revising..." then final result. Progress indicators without the full back-and-forth
- **D-07:** Final plan includes a **summary of changes** showing what Codex's review improved vs the original draft. User sees the value of the review loop

### Carrying Forward from Prior Phases
- Phase 1 D-01: Moderate guardrails — Codex can make minor judgment calls
- Phase 1 D-08: Attributed results — user knows which model did the work
- Phase 2 D-07: Plan-phase blocked until Codex reviews (Phase 3 upgrades this to multi-round)
- Phase 2 D-12: Review depth varies by task type

### Claude's Discretion
- Handoff spec format: Claude designs the typed handoff spec structure (fields, sections, `decisions_not_taken` format) based on what downstream execution needs
- Review state persistence: Claude implements `.planning/review-state.json` round counter and state tracking
- Adversarial prompt design: Claude crafts the adversarial review prompt that tells Codex to poke holes and find edge cases
- Low-confidence detection: Claude defines signals that trigger escalation from GPT-5.4-mini to Opus
- Superpowers skill modification approach: Claude determines how to modify `dispatching-parallel-agents` skill for model routing (after verifying skill symlink path)

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Research & Benchmarks
- `docs/research/opus-vs-codex-model-comparison.md` — Cross-model review loop effectiveness data; multi-round review benchmarks; GPT-5.4-mini capabilities for hypothesis testing
- `codex-claude-code-power-user-research.md` — Power user patterns for review loops and Codex integration

### Project Configuration
- `CLAUDE.md` — Hook events, ALLOW/BLOCK protocol, "What NOT to Use" constraints (no Agents SDK, no LangChain)
- `.planning/REQUIREMENTS.md` — Phase 3 requirements: REVW-03, REVW-04, REVW-05, REVW-06, SPWR-01, SPWR-02, SPWR-03

### Prior Phase Context
- `.planning/phases/01-foundation/01-CONTEXT.md` — Routing, AGENTS.md, tool-call detection, attributed results
- `.planning/phases/02-review-gate-gsd-integration/02-CONTEXT.md` — Review gate, feedback visibility, GSD checkpoints, review scope decisions

### Superpowers Internals (verify before implementing)
- Superpowers `dispatching-parallel-agents` skill — the skill to modify for model routing (symlink path flagged in STATE.md — must verify before modifying)
- Superpowers plan review skill — verify current structure before integrating the Opus-Codex review loop

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- **Phase 1 + 2 infrastructure** (once built): Codex invocation wrapper, review gate, token logging, routing logic — Phase 3 extends with multi-round and Superpowers integration
- **OpenAI Node.js SDK** (openai 6.33.0) — for GPT-5.4-mini API calls in Superpowers routing
- **Codex CLI** at `~/.npm-global/bin/codex` v0.118.0 — for review loop execution

### Established Patterns
- Phase 2's single-pass review becomes the first round of the multi-round loop
- Token logging already captures review events (Phase 2 D-06) — extend for multi-round tracking
- Attributed results pattern (Phase 1 D-08) applies to loop milestone updates

### Integration Points
- GSD plan-phase workflow: upgrade single-pass review to multi-round loop
- GSD task-level plans: add review loop trigger
- Superpowers `dispatching-parallel-agents`: add model routing logic
- Superpowers plan review: integrate same loop as GSD
- `.planning/review-state.json`: new file for round counter persistence

</code_context>

<specifics>
## Specific Ideas

- **Adversarial round is distinct** — the user specifically wants the second Codex review to be adversarial (poking holes, finding edge cases), not just another constructive review. The prompt for round 2 should be explicitly different from round 1.

</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope.

</deferred>

---

*Phase: 03-plan-review-loop-superpowers*
*Context gathered: 2026-04-02*
