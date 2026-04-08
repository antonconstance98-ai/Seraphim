# Phase 14 — Review Handoff Spec

**Reviewed:** 2026-04-03T21:43:12.277Z
**Rounds completed:** 2
**Early exit:** no
**Model authority:** Opus 4.6 (final authority per D-03)

## Plan Changes from Review

[PLAN 01] [SEVERITY: HIGH] Missing verification commands. The plan requires behavior changes in `codex-token-logger.js` and `codex-cost-reporter.js`, but the visible plan does not specify concrete commands to validate: v2.0 field pass-through, falsy-value preservation, backward compatibility with old log entries, three-model breakdown output, updated summary labels, and `fallbackCount`/`fallbackCost` in the return object.

[PLAN 01] [SEVERITY: MEDIUM] Requirements coverage is incomplete or not traceable. `requirements` lists `D-03` through `D-07`, but the visible task mapping only explicitly ties Task 1 to `D-04` and `D-05`; an executor would need clarification on which concrete task or verification step satisfies `D-03`, `D-06`, and `D-07`.

[PLAN 01] [SEVERITY: MEDIUM] Task actions are partly vague for the reporter changes. The token-logger edit is precisely specified, but the cost-reporter work, as shown, does not include explicit implementation steps for where `fallbackCount`/`fallbackCost` are computed, where the “Three-Model Breakdown” section is inserted, or how the return object and summary labels should be verified end-to-end.

## Adversarial Review

[CONCERN] [SEVERITY: HIGH] {The plan assumes `codex-token-logger.js` always receives a parsed `codexResult.tokens` object. If any producer emits a partial `[CODEX_RESULT]` marker without `tokens`, the existing `codexResult.tokens.input_tokens` access will throw before the new fields are written. That is a production silent-drop risk disguised as “backward compatible.”}

[CONCERN] [SEVERITY: HIGH] {“Old log entries without v2.0 fields still parse without errors” is not the same as “old malformed entries still parse.” The reporter change increases dependence on historical log shape, but the plan does not require defensive parsing for missing `cost_usd`, missing `model`, missing `source`, or non-numeric token fields. One bad JSONL line at 2 AM can poison the whole report.}

[CONCERN] [SEVERITY: HIGH] {The fallback definition relies on exact string equality for both `source === 'api-fallback'` and `model === 'minimax-m2.7'`. That is brittle. Any future producer emitting `api_fallback`, `fallback`, `minimax`, or a versioned MiniMax model silently disappears from fallback counts while still being charged in actual cost. The report will look “correct” and be wrong.}

[CONCERN] [SEVERITY: HIGH] {The plan explicitly says the token logger pass-through is for future fields emitted by “other Phase 9-12 hooks,” but it does not verify those hooks actually emit fields in the same shape or top-level location. If one producer nests `compression` or uses a different key, the logger technically satisfies the plan while production metadata capture remains broken.}

[CONCERN] [SEVERITY: HIGH] {The cost reporter is required to show “Codex cost, MiniMax cost, Opus baseline,” but the plan does not define whether “Codex cost” excludes fallback MiniMax records from `actualCost` or whether “Actual Cost” equals Codex+MiniMax. That ambiguity is the simplest way an executor ships a mathematically inconsistent table that still passes superficial review.}

[CONCERN] [SEVERITY: HIGH] {“Opus baseline (labeled as counterfactual)” is underspecified. If the executor computes Opus baseline for all records including MiniMax fallback executions, the number becomes a fantasy scenario where Opus handled calls Codex never completed. If they exclude fallback events, the baseline no longer matches total call count. Either way, the report can satisfy the wording while being analytically bogus.}

[CONCERN] [SEVERITY: HIGH] {The plan assumes `computeCost(tokens, model)` is safe for all historical records because MiniMax pricing exists. But the same helper falls back unknown models to `gpt-5.4`. That means bad model names produce valid-looking but incorrect costs instead of hard failures. This is a silent data integrity bug, not resilience.}

[CONCERN] [SEVERITY: HIGH] {No requirement forces the reporter to validate that `cost_usd` on stored records matches recomputation from tokens. If historical records were logged before pricing fixes or with stale prices, the reporter may aggregate a corrupted ledger forever. “Three-model breakdown” can be technically satisfied using already-wrong numbers.}

[CONCERN] [SEVERITY: MEDIUM] {The plan says “Do NOT change the existing record fields,” which strongly nudges an executor to append v2.0 fields blindly without checking serialization, log rotation, or downstream parsers. The easiest misinterpretation is to treat this as a pure write-side change and skip read-side compatibility testing entirely.}

[CONCERN] [SEVERITY: MEDIUM] {The object mutation instruction is hyper-specific about placement, but not about ordering guarantees in JSON output. Any downstream tooling that implicitly relies on field order in JSONL diffs or grep patterns may become harder to use even if nothing “breaks” formally. This is the kind of operational paper cut that shows up only during incidents.}

[CONCERN] [SEVERITY: MEDIUM] {The plan treats `false` and `0` preservation as important, but ignores other dangerous falsy/edge values: empty string, `null`, `NaN`, negative numbers. An executor can correctly implement `!== undefined` and still log nonsense values that later render as valid metadata.}

[CONCERN] [SEVERITY: MEDIUM] {The fallback event count is specified, but deduplication is not. If a failed handoff retries and logs multiple `api-fallback` entries for one user-visible task, the report will overcount “events” and overstate fallback spend. The system can be technically compliant and practically misleading.}

[CONCERN] [SEVERITY: MEDIUM] {The plan assumes one record equals one call. In real hooks pipelines, a single task can generate multiple records across review rounds, compression, retries, or partial failures. Renaming the summary label from “Total Codex Calls” to “Total Calls” hides this model mismatch instead of fixing it.}

[CONCERN] [SEVERITY: MEDIUM] {The instructions reference exact line numbers and code snippets from files under `~/.claude/hooks`, but those are user-home files outside the repo plan artifact. If the local environment drifts even slightly, the executor may patch the wrong area or duplicate logic. The plan is fragile to file version skew.}

[CONCERN] [SEVERITY: MEDIUM] {The task text is truncated. That alone creates a failure mode where the executor implements only the visible part and misses hidden acceptance criteria, especially around tests or return-object shape. A half-visible spec is an invitation for false completion.}

[CONCERN] [SEVERITY: MEDIUM] {The reporter return object is extended with `fallbackCount` and `fallbackCost`, but the plan does not require any caller, renderer, or template to consume them. That makes it easy to “return” the values and never surface them anywhere users actually see. Technically done, practically useless.}

[CONCERN] [SEVERITY: MEDIUM] {The plan says the primary value “RIGHT NOW” is model/source logging already sufficient for reporting, which invites an executor to deprioritize or omit verification of the new v2.0 fields. That is how future-facing compatibility features get merged untested and rot immediately.}

[CONCERN] [SEVERITY: MEDIUM] {Security and integrity risk: the logger is instructed to pass through arbitrary metadata fields from hook output into persistent JSONL without any schema validation. If upstream hooks ever emit oversized strings, unexpected objects, or user-derived values, you have a log poisoning vector and potentially broken downstream report generation.}

[CONCERN] [SEVERITY: LOW] {The plan assumes cost formatting and table labels can be changed without breaking snapshot tests, downstream markdown scrapers, or human workflows that grep for the old strings. Label-only changes are the classic “nonfunctional” break that nobody notices until automation fails.}

[CONCERN] [SEVERITY: LOW] {“Three-Model Breakdown” is required as a literal contains-check. An executor can satisfy that by adding a header while the actual numbers are incomplete, mislabeled, or missing percentage context. This is a weak artifact check that rewards cosmetic compliance.}

[CONCERN] [SEVERITY: LOW] {The plan focuses on backward compatibility with missing v2.0 fields but ignores forward compatibility with new fields beyond the listed five. The approach hardcodes a moving schema and guarantees another plan will be needed for every metadata addition. That is operationally brittle.}

[CONCERN] [SEVERITY: LOW] {At 2 AM, the likely failure is not a crash but a believable lie: the report generates, numbers add up, and no one notices that fallback spend is undercounted because one producer changed a string constant. This plan has too many exact-match assumptions and not enough validation to catch that class of failure.}

## Decisions Not Taken

| Issue | Raised by | Round | Reason Not Implemented |
|-------|-----------|-------|------------------------|
| To be populated after Opus final revision | — | — | — |

_Opus retains final authority after all review rounds. Codex concerns that Opus did not address are logged here with reasons. Per D-03, Codex concerns not acted on are not defects — they represent deliberate trade-off decisions by the architect._

## Review Verdict

BLOCKED_HIGH_SEVERITY
