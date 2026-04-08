# Phase 10 — Codex Plan Review

**Reviewed:** 2026-04-03T19:30:14.974Z
**Model:** gpt-5.4
**Plans reviewed:** 10-01-PLAN.md, 10-02-PLAN.md
**Review type:** Multi-round (constructive + adversarial)

## Findings

=== Round 1 (constructive) ===
[PLAN 01] [SEVERITY: HIGH] Verification is underspecified. The plan defines behavioral tests, but it does not provide executable verification commands for them, such as the exact `node`, test runner, or hook invocation commands needed to prove `reasoning_split`, fallback behavior, state migration, and token logging work end to end.

[PLAN 01] [SEVERITY: HIGH] Several task actions are still too vague for an executor. The plan says to implement D-08 fallback, backward compatibility for old `review-state.json`, and per-model token logging, but it does not specify the exact control flow changes, the state schema/defaulting behavior on read, or the concrete logging fields/expected values to update.

[PLAN 01] [SEVERITY: MEDIUM] Requirement coverage is incomplete in the task breakdown. D-04 and D-08 are explicitly described, but the plan does not clearly map task actions to D-01, D-03, D-05, D-06, and D-07, so an executor cannot tell which code changes satisfy each requirement or whether any are left out.

[PLAN 01] [SEVERITY: MEDIUM] The verification section misses commands for backward-compatibility and resume behavior. Because one must-have is that old `review-state.json` files without a `model` field resume without errors, the plan should include a concrete command or fixture-based replay to validate loading and resuming an old state file.

[PLAN 01] [SEVERITY: MEDIUM] The verification section misses commands for the “MiniMax fails OR returns empty text” branch. The plan states the fallback requirement, but it does not define how to simulate API failure, empty-text success, and the expected observable output proving Codex adversarial fallback actually ran.

[PLAN 01] [SEVERITY: LOW] The plan references “Test 1” through “Test 8” inside the task, but it does not say whether those are to be implemented as automated tests, run manually, or just used as acceptance criteria. An executor would need clarification to avoid skipping durable test coverage.

---

=== Round 2 (adversarial) ===
[CONCERN] [SEVERITY: HIGH] {The plan assumes MiniMax will actually return a usable "reasoning chain" in API output and that exposing it is stable behavior. That is an unsafe product assumption, not an implementation fact. Providers regularly suppress, redact, rename, or gate internal reasoning fields. If MiniMax returns only answer text, the headline objective "preserving full reasoning chain transparency" is practically false even if the integration technically succeeds.}

[CONCERN] [SEVERITY: HIGH] {The fallback rule is under-specified for partial failures. "If MiniMax fails OR returns empty text" is too narrow. The production failure mode is non-empty but useless output: whitespace, malformed think tags, repeated prompt, JSON blob, safety refusal, or a truncated answer. That technically satisfies "not empty text" and prevents fallback while still breaking Round 2.}

[CONCERN] [SEVERITY: HIGH] {The plan relies on a brittle assumption that unknown request fields are always passed through cleanly by the OpenAI SDK path and only validated server-side. Even if the current SDK implementation does this, wrappers, retries, middleware, or future SDK changes can normalize or strip fields. The whole "reasoning_split opt-in" path can silently no-op in production with no detection.}

[CONCERN] [SEVERITY: HIGH] {The cost logging requirement is technically weak and practically misleading. Reusing `computeCodexCostStrict` for `minimax-m2.7` assumes MiniMax token accounting semantics match the existing pricing model closely enough to be meaningful. Different providers often count cached tokens, reasoning tokens, and output tokens differently. Your logs can look precise while being economically wrong.}

[CONCERN] [SEVERITY: HIGH] {The plan caps Round 1 findings at 4000 characters and treats that as sufficient context transfer. That is an arbitrary truncation strategy with no guarantee that the most critical findings survive. The simplest production failure is the adversarial round missing the one key issue because truncation dropped it, while the system still claims Round 1 context was passed.}

[CONCERN] [SEVERITY: HIGH] {Backward compatibility for `review-state.json` is oversimplified. Defaulting missing `model` to `gpt-5.4` handles only one schema drift. It does nothing for mixed old/new records, partially written state files, interrupted writes, stale round status, or resumed executions where Round 2 was previously started under different assumptions. Resume can silently replay or misattribute rounds.}

[CONCERN] [SEVERITY: MEDIUM] {The plan treats "reasoning_content" and "reasoning_details" as the relevant alternate response fields, but that is still guesswork over an unstable provider contract. If MiniMax uses a nested shape, content parts array, or future renamed field, the code will degrade to content-only and the stated objective will quietly fail.}

[CONCERN] [SEVERITY: MEDIUM] {JSON-stringifying object-valued reasoning fields is technically safe but practically broken. If the field contains a structured trace, large nested metadata, or tool artifacts, the output becomes unreadable noise inside `<think>` tags. That can swamp the actual review and degrade downstream consumers or handoff summaries.}

[CONCERN] [SEVERITY: MEDIUM] {The plan assumes plain-text composition with think tags is harmless. It is not. Injecting provider reasoning into the visible review text can break any downstream parser, summarizer, or UI that expects findings-only output. "Works as text" is not the same as "works in the workflow."}

[CONCERN] [SEVERITY: MEDIUM] {There is an unchallenged assumption that MiniMax produces genuinely different adversarial reasoning than Codex in a way that improves review quality. That is the core justification for the change, but the plan offers no validation mechanism. The simplest real-world outcome is higher complexity, more failure modes, and no meaningful diversity gain.}

[CONCERN] [SEVERITY: MEDIUM] {The retry story is shallow. The review feedback mentions retrying without `reasoning_split` if rejected, but that only covers one failure class. It does not address timeouts after partial output, provider-side 200 responses with malformed bodies, rate limiting under a different error shape, or retries that duplicate billable requests.}

[CONCERN] [SEVERITY: MEDIUM] {The plan claims "Round 2 gracefully degrades to content-only if reasoning fields absent," but that is only graceful from an exception-handling perspective. From a product perspective, it silently violates the stated purpose. A system that claims transparent reasoning while quietly dropping it is practically broken even if it does not crash.}

[CONCERN] [SEVERITY: MEDIUM] {The requirement "MiniMax reasoning chain appears in Round 2 text when API provides it" ignores size explosion. A long reasoning trace can dominate the round record, exceed state or summary expectations, make handoff artifacts unusable, or trigger token/cost spikes on any later processing of the saved review.}

[CONCERN] [SEVERITY: LOW] {The tests described focus on response-shape normalization but miss output correctness at the workflow boundary. They can all pass while the actual review file is unreadable, state transitions are wrong, fallback never triggers when it should, or token logs are misleading.}

[CONCERN] [SEVERITY: LOW] {The requirement "contains `reasoning_split`" in the artifact definition is weak to the point of being cosmetic. A string match proves almost nothing about whether the field is attached in the right request path, under the right conditions, or observed by the upstream API.}

[CONCERN] [SEVERITY: LOW] {The plan assumes the existing "extractCodexText" isolation is sufficient because MiniMax returns plain text. That overlooks any place in the rest of the pipeline that still implicitly assumes Codex-style structure, model names, or source metadata. Cross-provider plumbing usually fails in the boring edges, not the main execution path.}

## Verdict

BLOCKED_HIGH_SEVERITY
