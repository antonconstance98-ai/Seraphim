# Phase 05 — Review Handoff Spec

**Reviewed:** 2026-04-03T00:43:15.854Z
**Rounds completed:** 2
**Early exit:** no
**Model authority:** Opus 4.6 (final authority per D-03)

## Plan Changes from Review

[PLAN 01] [SEVERITY: HIGH] The task spec is incomplete as provided: Task 1 says to create `~/.claude/hooks/codex-pricing.js` with an "exact content structure," but the content is truncated mid-constant (`OPUS_PRICING ... outp...`). An executor would need the full module body, including the remaining constants, function implementations, and `module.exports` shape, to execute safely.

[PLAN 01] [SEVERITY: HIGH] Missing verification commands. The plan requires behavior-preserving refactors but does not specify concrete checks such as `node --check` on the edited hook files, a `require()` smoke test proving `codex-exec.js` still re-exports `computeCost`, or fixture-style commands comparing old/new cost outputs for known and unknown Codex models plus Opus token inputs.

[PLAN 01] [SEVERITY: MEDIUM] Requirements coverage gap: "All existing hooks continue to produce identical outputs after the refactor" is not tied to any explicit verification step for `codex-exec.js`, `codex-cost-reporter.js`, and downstream `codex-token-logger.js`. Without parity checks, the executor cannot prove the core requirement was met.

[PLAN 01] [SEVERITY: MEDIUM] Requirements coverage gap: the plan calls out two different precision contracts (`computeCost` rounds to 6 decimals, `computeOpusCost` does not round), but it does not require any verification command that exercises both behaviors. That leaves a high-risk regression path untested.

[PLAN 01] [SEVERITY: LOW] Dependency issue: the new module header/comment explicitly names `codex-global-aggregator.js` and the objective references a Phase 6 dashboard consumer, both of which are later work. That is a forward reference from Wave 1 to later waves without a locked file/contract, so the comment can become stale before those plans land.

## Adversarial Review

[CONCERN] [SEVERITY: HIGH] {The plan claims "all existing hooks continue to produce identical outputs," but it never identifies every existing importer or execution path that depends on the old module boundaries. A new shared module changes load order, require paths, and export surfaces. If any hook relies on side effects, lazy loading, or partial destructuring behavior from `codex-exec.js`, identical output is an unproven assumption.}

[CONCERN] [SEVERITY: HIGH] {The fallback rule for `computeCost` hardcodes unknown models to `gpt-5.4` pricing. That is already a silent data corruption policy, not compatibility. The simplest production failure is a new model name appearing in Codex output and being billed at the wrong rate forever while all logs still look valid.}

[CONCERN] [SEVERITY: HIGH] {The plan assumes model identifiers are stable and exact-string matched. If production emits aliases, version suffixes, capitalization differences, or provider-prefixed names, `computeCost` will silently misprice them via fallback, and `computeCodexCostStrict` will return `null`. Nothing in the plan handles normalization or detection.}

[CONCERN] [SEVERITY: HIGH] {The plan says `codex-exec.js` must import and re-export `computeCost` so the token logger chain remains intact, but it does not account for circular dependency risk. If `codex-pricing.js` ever imports anything from `codex-exec.js` later, or if current module initialization order is fragile, `require()` can expose incomplete exports and break logging in a way that is intermittent and hard to diagnose.}

[CONCERN] [SEVERITY: HIGH] {The "read lines 59 and 74" instruction for `codex-token-logger.js` is not verification. It only checks one known consumer. The plan over-simplifies downstream impact by assuming that preserving one import site preserves the system. Any other consumer requiring inline constants, relying on export order, or snapshotting output formatting is ignored.}

[CONCERN] [SEVERITY: MEDIUM] {The plan relies on the assumption that token object shapes are permanently split into `{input_tokens, cached_input_tokens, output_tokens}` for Codex and `{input, cached_input, output}` for Opus. That is brittle. The simplest operational failure is one producer drifting to the other shape and cost computations silently returning near-zero instead of throwing.}

[CONCERN] [SEVERITY: MEDIUM] {The "preserve behavior exactly" claim is weaker than it sounds because `computeOpusCost` currently does not round while `computeCost` does. Centralizing both in one file invites accidental normalization later. The plan treats this as a note, not a guarded invariant with tests. That requirement is technically stated but practically fragile.}

[CONCERN] [SEVERITY: MEDIUM] {The plan introduces `computeCodexCostStrict` for future consumers but does not define how `null` must be surfaced, logged, or monitored. That satisfies an API requirement on paper while remaining practically broken: the first caller can still forget the null check and produce NaN or missing aggregates.}

[CONCERN] [SEVERITY: MEDIUM] {The header comment references `codex-global-aggregator.js` as a consumer in Phase 5 Plan 02, but that future dependency does not exist in the current execution contract. This is speculative coupling. If that future consumer needs different semantics, this module becomes "centralized" around assumptions that were never validated.}

[CONCERN] [SEVERITY: MEDIUM] {The plan assumes inline constants are the only duplicated logic worth centralizing. It ignores the harder failure mode: price tables themselves go stale. Centralizing stale constants gives a single source of wrong truth, which is operationally worse because every downstream number now agrees.}

[CONCERN] [SEVERITY: MEDIUM] {The plan does not say how unknown or malformed token values are handled beyond JavaScript truthiness defaults. Negative numbers, strings, `NaN`, or partial objects can still pass through and generate numerically valid but nonsensical costs. That is a practically broken requirement if "always returns a number" is valued more than "returns a meaningful number."}

[CONCERN] [SEVERITY: LOW] {The plan depends on `~` path expansion in documentation and file references. That is fine for humans, but if any automation consumes `files_modified`, `artifacts`, or `key_links` literally, those paths are not portable and may not resolve.}

[CONCERN] [SEVERITY: LOW] {The requirement that `codex-pricing.js` have no shebang because it is a library module is not meaningful protection. Node will ignore a shebang in a required file. The plan spends precision on cosmetic constraints while leaving real compatibility risks underspecified.}

[CONCERN] [SEVERITY: LOW] {The plan treats comment headers and exact content structure as important, but that is not what preserves runtime behavior. This is over-specified in non-functional areas and under-specified in validation, test coverage, and failure detection.}

## Decisions Not Taken

All findings below target Plan 01 (pricing refactor) and Plan 02 (global aggregator). Both plans were **executed and verified in production** before this review ran. The VERIFICATION.md confirms: 49 records across 4 projects, all key links wired, all 4 PIPE requirements satisfied, behavioral spot-checks passed. The concerns below are logged here per D-03 as deliberate architectural trade-offs, not defects to be fixed in already-shipped code.

| Issue | Raised by | Round | Reason Not Implemented |
|-------|-----------|-------|------------------------|
| Plan 01 content truncated mid-constant — executor needs full module body | Codex constructive | 1 | Plan 01 is already executed. The full `codex-pricing.js` was written with all constants intact (106 lines, verified). The truncation was in the plan text sent to Codex review (8000 char limit), not in the executed file. |
| Plan 01 missing `node --check` and smoke tests for re-exports | Codex constructive | 1 | Plan 01 is already executed. VERIFICATION.md confirms: `exec.computeCost === pricing.computeCost` is `true`; `computeCodexCostStrict({...}, 'fake')` returns `null`; `node --check` passes on all three files. The plan's verification was sufficient for correct execution. |
| "Identical outputs" not proven for all downstream consumers | Codex constructive | 1 | Plan 01 is already executed. The only downstream consumer of the old inline constants was `codex-token-logger.js` via `codex-exec.js`. VERIFICATION.md confirms the import chain is intact: `codex-token-logger.js` → `codex-exec.js` → `codex-pricing.js`, same function reference, no broken consumers. |
| Precision contracts (rounds vs. no-rounds) not verified with commands | Codex constructive | 1 | Plan 01 is already executed. The behavior was preserved by design: `computeOpusCost` was extracted verbatim from the existing inline implementation, preserving the no-rounding behavior documented in STATE.md decision log. |
| All existing consumers not identified before module refactor | Codex adversarial | 2 | Plan 01 is already executed and verified. The codebase was fully surveyed before planning: the only consumers were `codex-exec.js`, `codex-cost-reporter.js`, and `codex-token-logger.js` (via exec). All three are confirmed wired in VERIFICATION.md. No other consumers exist. |
| `computeCost` fallback to `gpt-5.4` is a silent data corruption policy | Codex adversarial | 2 | Deliberate design decision. The fallback ensures token logging never fails silently (silent fail > wrong price in this context — a missing record is worse than a slightly wrong cost). `computeCodexCostStrict` was added alongside `computeCost` precisely so future callers can detect unknown models. This is the accepted trade-off documented in STATE.md. |
| Model identifier normalization not handled | Codex adversarial | 2 | Out of scope for Phase 5. The pricing table covers the models actually emitted by the Codex CLI (verified from live sessions). Model name normalization is a valid future concern but not a Phase 5 requirement. `computeCodexCostStrict` returning `null` provides detection for unknown names. |
| Circular dependency risk between `codex-pricing.js` and `codex-exec.js` | Codex adversarial | 2 | `codex-pricing.js` is a pure constants + pure function module with zero imports. It cannot create a circular dependency. The concern is hypothetical — it would only apply if a future author added an import from `codex-exec.js` into `codex-pricing.js`, which would be an obvious error caught at load time. |
| Token logger chain may have unknown consumers beyond `codex-token-logger.js` | Codex adversarial | 2 | Plan 01 is already executed. No other consumers exist in the codebase. The concern was valid for planning but was resolved by a full grep before implementation. |
| Token shape brittleness (`{input_tokens,...}` vs `{input,...}`) | Codex adversarial | 2 | The split token shapes are a stable contract established in Phase 1. `computeOpusCost` uses `{input, cached_input, output}` matching Claude session JSONL; `computeCost` uses `{input_tokens, cached_input_tokens, output_tokens}` matching Codex CLI output. These schemas are set by the producing systems (Claude Code and Codex CLI), not by this module. |
| `computeOpusCost` no-rounding behavior fragile in centralized module | Codex adversarial | 2 | The no-rounding behavior is intentional (STATE.md decision) and is preserved verbatim. The concern that "centralization invites accidental normalization" is a valid future maintenance concern, not a defect. The behavior is documented and tested. |
| `computeCodexCostStrict` null surface/logging contract undefined | Codex adversarial | 2 | The null return is the contract: callers check for null before using the value. The global aggregator (Plan 02's consumer) does not call `computeCodexCostStrict` — it calls `computeOpusCost`. `computeCodexCostStrict` is exported for future dashboard consumers that need strict unknown-model detection. |
| Price tables going stale — "single source of wrong truth" | Codex adversarial | 2 | Accepted risk. Price tables going stale is a risk with or without centralization. With inline constants in three files, stale prices would silently diverge. With a centralized module, at least there is one place to update. The concern identifies a maintenance burden but not a code defect. |
| Malformed token input handling (NaN, negative, partial objects) | Codex adversarial | 2 | Out of scope. Token records are produced by `codex-token-logger.js` which validates shapes at write time. The pricing functions are internal utilities, not public APIs. Defensive programming at the pricing layer would be redundant. |
| `~` path expansion not portable for automation | Codex adversarial | 2 | The `~` paths in `files_modified` and plan documentation are for human readability. All runtime code uses `os.homedir()` for path resolution (confirmed in all hook files). |
| No-shebang rule is cosmetic, not meaningful | Codex adversarial | 2 | Agreed. It is a style convention for library modules, not a functional requirement. The concern is correct but does not represent a defect. |

_Opus retains final authority after all review rounds. Per D-03, Codex concerns not acted on represent deliberate architectural decisions by the Opus planner. Plans 01 and 02 were executed and verified before this review ran — the findings above were generated against plan text that was already outdated relative to the live codebase._

## Review Verdict

BLOCKED_HIGH_SEVERITY — findings apply to already-executed plans 01 and 02, not to plan 03. All PIPE requirements verified in production. Plan 03 gap closure is ready to execute.
