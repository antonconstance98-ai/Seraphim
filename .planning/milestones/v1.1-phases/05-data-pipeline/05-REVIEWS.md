# Phase 05 — Codex Plan Review

**Reviewed:** 2026-04-03T00:43:15.854Z
**Model:** gpt-5.4
**Plans reviewed:** 05-01-PLAN.md, 05-02-PLAN.md, 05-03-PLAN.md
**Review type:** Multi-round (constructive + adversarial)

## Findings

=== Round 1 (constructive) ===
[PLAN 01] [SEVERITY: HIGH] The task spec is incomplete as provided: Task 1 says to create `~/.claude/hooks/codex-pricing.js` with an “exact content structure,” but the content is truncated mid-constant (`OPUS_PRICING ... outp...`). An executor would need the full module body, including the remaining constants, function implementations, and `module.exports` shape, to execute safely.

[PLAN 01] [SEVERITY: HIGH] Missing verification commands. The plan requires behavior-preserving refactors but does not specify concrete checks such as `node --check` on the edited hook files, a `require()` smoke test proving `codex-exec.js` still re-exports `computeCost`, or fixture-style commands comparing old/new cost outputs for known and unknown Codex models plus Opus token inputs.

[PLAN 01] [SEVERITY: MEDIUM] Requirements coverage gap: “All existing hooks continue to produce identical outputs after the refactor” is not tied to any explicit verification step for `codex-exec.js`, `codex-cost-reporter.js`, and downstream `codex-token-logger.js`. Without parity checks, the executor cannot prove the core requirement was met.

[PLAN 01] [SEVERITY: MEDIUM] Requirements coverage gap: the plan calls out two different precision contracts (`computeCost` rounds to 6 decimals, `computeOpusCost` does not round), but it does not require any verification command that exercises both behaviors. That leaves a high-risk regression path untested.

[PLAN 01] [SEVERITY: LOW] Dependency issue: the new module header/comment explicitly names `codex-global-aggregator.js` and the objective references a Phase 6 dashboard consumer, both of which are later work. That is a forward reference from Wave 1 to later waves without a locked file/contract, so the comment can become stale before those plans land.

---

=== Round 2 (adversarial) ===
[CONCERN] [SEVERITY: HIGH] {The plan claims “all existing hooks continue to produce identical outputs,” but it never identifies every existing importer or execution path that depends on the old module boundaries. A new shared module changes load order, require paths, and export surfaces. If any hook relies on side effects, lazy loading, or partial destructuring behavior from `codex-exec.js`, identical output is an unproven assumption.}

[CONCERN] [SEVERITY: HIGH] {The fallback rule for `computeCost` hardcodes unknown models to `gpt-5.4` pricing. That is already a silent data corruption policy, not compatibility. The simplest production failure is a new model name appearing in Codex output and being billed at the wrong rate forever while all logs still look valid.}

[CONCERN] [SEVERITY: HIGH] {The plan assumes model identifiers are stable and exact-string matched. If production emits aliases, version suffixes, capitalization differences, or provider-prefixed names, `computeCost` will silently misprice them via fallback, and `computeCodexCostStrict` will return `null`. Nothing in the plan handles normalization or detection.}

[CONCERN] [SEVERITY: HIGH] {The plan says `codex-exec.js` must import and re-export `computeCost` so the token logger chain remains intact, but it does not account for circular dependency risk. If `codex-pricing.js` ever imports anything from `codex-exec.js` later, or if current module initialization order is fragile, `require()` can expose incomplete exports and break logging in a way that is intermittent and hard to diagnose.}

[CONCERN] [SEVERITY: HIGH] {The “read lines 59 and 74” instruction for `codex-token-logger.js` is not verification. It only checks one known consumer. The plan over-simplifies downstream impact by assuming that preserving one import site preserves the system. Any other consumer requiring inline constants, relying on export order, or snapshotting output formatting is ignored.}

[CONCERN] [SEVERITY: MEDIUM] {The plan relies on the assumption that token object shapes are permanently split into `{input_tokens, cached_input_tokens, output_tokens}` for Codex and `{input, cached_input, output}` for Opus. That is brittle. The simplest operational failure is one producer drifting to the other shape and cost computations silently returning near-zero instead of throwing.}

[CONCERN] [SEVERITY: MEDIUM] {The “preserve behavior exactly” claim is weaker than it sounds because `computeOpusCost` currently does not round while `computeCost` does. Centralizing both in one file invites accidental normalization later. The plan treats this as a note, not a guarded invariant with tests. That requirement is technically stated but practically fragile.}

[CONCERN] [SEVERITY: MEDIUM] {The plan introduces `computeCodexCostStrict` for future consumers but does not define how `null` must be surfaced, logged, or monitored. That satisfies an API requirement on paper while remaining practically broken: the first caller can still forget the null check and produce NaN or missing aggregates.}

[CONCERN] [SEVERITY: MEDIUM] {The header comment references `codex-global-aggregator.js` as a consumer in Phase 5 Plan 02, but that future dependency does not exist in the current execution contract. This is speculative coupling. If that future consumer needs different semantics, this module becomes “centralized” around assumptions that were never validated.}

[CONCERN] [SEVERITY: MEDIUM] {The plan assumes inline constants are the only duplicated logic worth centralizing. It ignores the harder failure mode: price tables themselves go stale. Centralizing stale constants gives a single source of wrong truth, which is operationally worse because every downstream number now agrees.}

[CONCERN] [SEVERITY: MEDIUM] {The plan does not say how unknown or malformed token values are handled beyond JavaScript truthiness defaults. Negative numbers, strings, `NaN`, or partial objects can still pass through and generate numerically valid but nonsensical costs. That is a practically broken requirement if “always returns a number” is valued more than “returns a meaningful number.”}

[CONCERN] [SEVERITY: LOW] {The plan depends on `~` path expansion in documentation and file references. That is fine for humans, but if any automation consumes `files_modified`, `artifacts`, or `key_links` literally, those paths are not portable and may not resolve.}

[CONCERN] [SEVERITY: LOW] {The requirement that `codex-pricing.js` have no shebang because it is a library module is not meaningful protection. Node will ignore a shebang in a required file. The plan spends precision on cosmetic constraints while leaving real compatibility risks underspecified.}

[CONCERN] [SEVERITY: LOW] {The plan treats comment headers and exact content structure as important, but that is not what preserves runtime behavior. This is over-specified in non-functional areas and under-specified in validation, test coverage, and failure detection.}

## Verdict

BLOCKED_HIGH_SEVERITY
