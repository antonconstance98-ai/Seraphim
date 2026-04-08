# Phase 10 — Review Handoff Spec

**Reviewed:** 2026-04-03T19:30:14.973Z
**Rounds completed:** 2
**Early exit:** no
**Model authority:** Opus 4.6 (final authority per D-03)
**Opus triage:** 2026-04-04

## Plan Changes from Review (Accepted)

The following Round 1 constructive findings were accepted and should be addressed during execution:

[PLAN 01] [SEVERITY: HIGH] Verification is underspecified. The plan defines behavioral tests, but it does not provide executable verification commands for them, such as the exact `node`, test runner, or hook invocation commands needed to prove `reasoning_split`, fallback behavior, state migration, and token logging work end to end.

[PLAN 01] [SEVERITY: HIGH] Several task actions are still too vague for an executor. The plan says to implement D-08 fallback, backward compatibility for old `review-state.json`, and per-model token logging, but it does not specify the exact control flow changes, the state schema/defaulting behavior on read, or the concrete logging fields/expected values to update.

[PLAN 01] [SEVERITY: MEDIUM] Requirement coverage is incomplete in the task breakdown. D-04 and D-08 are explicitly described, but the plan does not clearly map task actions to D-01, D-03, D-05, D-06, and D-07, so an executor cannot tell which code changes satisfy each requirement or whether any are left out.

[PLAN 01] [SEVERITY: MEDIUM] The verification section misses commands for backward-compatibility and resume behavior. Because one must-have is that old `review-state.json` files without a `model` field resume without errors, the plan should include a concrete command or fixture-based replay to validate loading and resuming an old state file.

[PLAN 01] [SEVERITY: MEDIUM] The verification section misses commands for the “MiniMax fails OR returns empty text” branch. The plan states the fallback requirement, but it does not define how to simulate API failure, empty-text success, and the expected observable output proving Codex adversarial fallback actually ran.

[PLAN 01] [SEVERITY: LOW] The plan references “Test 1” through “Test 8” inside the task, but it does not say whether those are to be implemented as automated tests, run manually, or just used as acceptance criteria. An executor would need clarification to avoid skipping durable test coverage.

## Adversarial Review — Opus Triage

Round 2 adversarial concerns were raised by Codex (gpt-5.4). Per D-03, Opus retains final authority. Each concern is triaged below as ACCEPT (incorporate into execution), ACKNOWLEDGED (valid observation, handled by existing design), or OVERRIDE (not actionable for this phase).

### Accepted (incorporate into plan)

[CONCERN] [SEVERITY: HIGH] Fallback rule under-specified for partial failures — “If MiniMax fails OR returns empty text” is too narrow; non-empty but useless output (whitespace, malformed tags, safety refusal) would bypass fallback.
**Action:** Widen fallback trigger to include content-quality heuristics (minimum length, non-whitespace ratio, absence of safety-refusal patterns).

[CONCERN] [SEVERITY: HIGH] Round 1 context truncation at 4000 chars has no priority ordering — critical findings could be dropped.
**Action:** Truncate by severity (HIGH first, then MEDIUM, then LOW) rather than position. Include a count of dropped findings if truncation occurs.

### Acknowledged (valid, handled by existing design)

[CONCERN] [SEVERITY: HIGH] MiniMax reasoning chain availability is an unstable provider contract.
**Why acknowledged, not blocked:** D-04 already specifies `reasoning_split: true` as opt-in via `extra_body`, and the plan already includes content-only degradation when reasoning fields are absent. The concern is valid but the mitigation is already designed. Execution should add a log line when degradation occurs.

[CONCERN] [SEVERITY: HIGH] `reasoning_split` passthrough via OpenAI SDK could break silently.
**Why acknowledged, not blocked:** Same mitigation — content-only degradation is the fallback. Execution should verify the field reaches the API during integration testing and log when it's stripped.

[CONCERN] [SEVERITY: HIGH] Cost logging with `computeCodexCostStrict` may not match MiniMax token semantics.
**Why acknowledged, not blocked:** Token counts are logged for relative comparison and budget awareness, not billing reconciliation. The function name will be updated to reflect multi-model use. Exact MiniMax pricing is documented in research and will be applied at the rate-per-token level.

[CONCERN] [SEVERITY: HIGH] Backward compatibility for `review-state.json` is oversimplified.
**Why acknowledged, not blocked:** This is a single-user local tool, not a distributed system. The schema has one new optional field (`model`). Defaulting to `gpt-5.4` for missing fields covers the only real migration case. Interrupted writes and mixed records are filesystem-level concerns already handled by the existing atomic-write pattern.

[CONCERN] [SEVERITY: MEDIUM] Reasoning field names are guesswork over an unstable contract.
**Why acknowledged, not blocked:** Phase 8 research (minimax-m2.7-synthesis.md §5) documents the actual field names. Content-only degradation handles any future rename.

[CONCERN] [SEVERITY: MEDIUM] JSON-stringifying object reasoning fields could produce unreadable output.
**Why acknowledged, not blocked:** Valid edge case. Execution should cap reasoning content at a reasonable size and add a truncation marker if exceeded.

[CONCERN] [SEVERITY: MEDIUM] Think tag injection could break downstream parsers.
**Why acknowledged, not blocked:** D-03 explicitly chose full transparency. REVIEWS.md is a human-readable artifact, not machine-parsed. The concern about downstream parsers applies to a pipeline that doesn't exist yet.

### Overridden (not actionable for this phase)

| Concern | Severity | Reason Overridden |
|---------|----------|-------------------|
| MiniMax may not produce genuinely different adversarial reasoning than Codex | MEDIUM | This is the core hypothesis of the phase. Validation comes from observing real outputs, not from pre-implementation proof. Deferred to post-ship evaluation. |
| Retry story is shallow (only covers `reasoning_split` rejection) | MEDIUM | Acceptable for v1. MiniMax retry complexity is bounded by the D-08 fallback — if anything goes wrong, Codex takes over. Hardening retries is a future improvement. |
| “Graceful degradation” silently violates stated purpose | MEDIUM | Degradation is logged and visible in REVIEWS.md output. The user can see when reasoning was unavailable. Silent from a crash perspective, transparent from a product perspective. |
| Reasoning trace size explosion in round records | MEDIUM | Addressed by the size-cap acknowledgment above. Not a blocker. |
| Tests miss output correctness at workflow boundary | LOW | Valid but LOW severity. Execution should include at least one end-to-end smoke test that checks the final REVIEWS.md output. |
| `reasoning_split` string match is cosmetic verification | LOW | Agreed it's weak in isolation, but it's one check among many. Not worth blocking on. |
| `extractCodexText` may have implicit Codex assumptions | LOW | Will be audited during execution. Low risk given the function is already isolated. |

## Review Verdict

**Raw reviewer verdict:** BLOCKED_HIGH_SEVERITY
**Opus authoritative verdict:** PASS_WITH_CHANGES

All Round 1 constructive findings are accepted and will be incorporated during execution. Of the 6 HIGH-severity adversarial concerns, 2 are accepted as plan changes, 4 are acknowledged with existing mitigations documented above. No concern represents an architectural flaw that would require replanning. The phase is cleared for execution with the accepted changes applied.

_Per D-03, Opus retains final authority. Overridden concerns are not defects — they represent deliberate trade-off decisions by the architect._
