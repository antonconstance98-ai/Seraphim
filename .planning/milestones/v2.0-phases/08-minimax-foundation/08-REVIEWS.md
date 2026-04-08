# Phase 08 — Codex Plan Review

**Reviewed:** 2026-04-03T18:00:51.175Z
**Model:** gpt-5.4
**Plans reviewed:** 08-01-PLAN.md, 08-02-PLAN.md, 08-03-PLAN.md
**Review type:** Multi-round (constructive + adversarial)

## Findings

=== Round 1 (constructive) ===
[PLAN 01] [SEVERITY: HIGH] Task 2 is incomplete/vague as written: the action block is truncated at `require('./codex-dashboard-gen...`, so an executor does not have the full implementation detail for how dashboard regeneration should be invoked, what export name/path to use, what CLI behavior/output is expected, or what exact command must be run to execute the migration.

[PLAN 01] [SEVERITY: HIGH] Missing verification commands for the migration outcomes required by D-09. Task 2 needs explicit commands to verify that `~/.claude/hooks/migrate-opus-pricing.js` was actually run, that `~/.claude/dashboard/global.jsonl` now contains recalculated `opus_baseline_usd` values, that malformed/blank lines were preserved unchanged, that dashboard HTML was regenerated, and that running the migration twice is idempotent.

[PLAN 01] [SEVERITY: MEDIUM] The requirement “All existing records in global.jsonl have `opus_baseline_usd` recalculated” is underspecified for records with valid JSON but no `tokens` field. The task says to preserve such records via `JSON.stringify(record)`, which does not clarify whether records that already contain stale `opus_baseline_usd` but lack `tokens` should remain unchanged or have that field removed/updated.

[PLAN 01] [SEVERITY: MEDIUM] The migration plan has a dependency ambiguity on Task 1 output. Task 2 says “read `codex-pricing.js` (after Task 1 -- confirm corrected `OPUS_PRICING`)” but its formula is hardcoded to `$5/$1.25/$25` instead of explicitly requiring reuse of `computeOpusCost` or `OPUS_PRICING`. That creates a drift risk if Task 1 is edited differently or future pricing changes occur.

[PLAN 01] [SEVERITY: LOW] Missing verification command for backward-compatibility of the non-migration output artifact. The plan says `dashboard.html` is regenerated, but no concrete command checks that the regenerated HTML reflects corrected savings percentages or that generation succeeded without error.

[PLAN 01] [SEVERITY: LOW] File conflict analysis is incomplete at the plan level. `~/.claude/hooks/codex-pricing.js` is modified in Task 1 and read as a prerequisite in Task 2, so the plan depends on strict sequential execution; it should state that these tasks are not parallelizable to avoid an executor or scheduler treating them as independent work.

---

=== Round 2 (adversarial) ===
I’m reviewing the plan against the actual hook code to find failure modes, not to validate intent. First step is to inspect the referenced pricing, aggregator, dashboard, and current data files for assumptions the plan may have missed.
The code already exposes one obvious fault line: the dashboard generator is asynchronous and the aggregator imports it synchronously. I’m checking whether the migration plan ignores async completion and whether it risks racing the live appender on `global.jsonl`.
I have enough context. The rest is about exact break modes: malformed JSON handling, numeric coercion, file rewrite races, and whether “dashboard regenerated” is only technically true while still leaving stale HTML in practice.
[CONCERN] [SEVERITY: HIGH] {The migration plan assumes `~/.claude/dashboard/global.jsonl` is quiescent while it is being rewritten. That assumption is false in the current system: [`codex-global-aggregator.js`](/home/alucard/.claude/hooks/codex-global-aggregator.js#L352) appends to the same file with `fs.appendFileSync`. A read-all, rewrite-all, temp-rename migration will silently drop any records appended after the migration starts and before rename completes. This is the simplest production failure mode: run migration during normal hook activity and lose data.}

[CONCERN] [SEVERITY: HIGH] {“Migration is idempotent” is only true in a toy environment. In production, the content will not be identical on a second run if new records arrive between runs, if line endings are normalized, or if the first run rewrites valid JSON lines through `JSON.stringify`. The plan overclaims byte-level idempotence without defining a quiescence requirement or a content-equivalence rule.}

[CONCERN] [SEVERITY: HIGH] {The plan says malformed and blank lines are preserved unchanged, but its own transform description uses `split('\n')` plus `join('\n')`. That does not preserve raw file content. It can change trailing newline presence, collapse CRLF to LF semantics, and alter the exact placement of terminal blank lines. “Unchanged” is technically false unless the script operates on raw line terminators and final EOF state.}

[CONCERN] [SEVERITY: MEDIUM] {The plan treats “valid JSON without `tokens`” as safe to preserve via `JSON.stringify(record)`. That is not preservation. It normalizes whitespace, removes original formatting, and may change key order relative to the source line. If downstream tooling or diffs expect minimal churn, the migration technically succeeds while practically rewriting unrelated content.}

[CONCERN] [SEVERITY: MEDIUM] {The recalculation formula assumes `tokens.input`, `tokens.cached_input`, and `tokens.output` are numeric or missing. That assumption can be false under schema drift. The dashboard generator already has a `safeNum` helper specifically because upstream numeric fields may arrive as strings or junk. The migration plan does not mention coercion or finite-number validation, so a single malformed numeric field can produce `NaN` and poison `opus_baseline_usd`.}

[CONCERN] [SEVERITY: MEDIUM] {The plan claims “all existing records in `global.jsonl` have `opus_baseline_usd` recalculated,” but its logic only touches valid JSON records with a `tokens` field. Any record with `tokens: null`, `tokens: []`, `tokens: "..."`, or partial-but-invalid token values will either get nonsense or be left structurally inconsistent. The requirement is broader than the actual safe handling.}

[CONCERN] [SEVERITY: MEDIUM] {The plan assumes dashboard regeneration is enough to reflect corrected savings percentages. In reality, [`generateDashboard`](/home/alucard/.claude/hooks/codex-dashboard-generator.js#L442) only rewrites HTML; it does not refresh Chart.js assets when required from another module. So “dashboard regenerated” can be technically true while the page is still practically broken or partially degraded if `assets/chart.min.js` is missing or stale.}

[CONCERN] [SEVERITY: MEDIUM] {The MiniMax addition is string-exact. The plan assumes the runtime model identifier is exactly `'minimax-m2.7'`. If the caller emits any variant, alias, or provider-prefixed name, [`computeCodexCostStrict`](/home/alucard/.claude/hooks/codex-pricing.js#L68) still returns `null`, and the new pricing support is technically added but practically nonfunctional for real traffic. The plan does not verify the producer side at all.}

[CONCERN] [SEVERITY: LOW] {The plan says “Do NOT change function signatures, function bodies, or exports,” but also requires a header comment update listing `minimax-exec.js` as a consumer. That comment is itself an assumption that the consumer already exists and imports this module. If it does not, the plan records a false dependency in source.}

[CONCERN] [SEVERITY: LOW] {The migration writes through a temp file and rename, but the plan says nothing about preserving original file mode. The replacement file inherits current umask, not necessarily the source file’s permissions. That is a small operational footgun the plan ignores.}

[CONCERN] [SEVERITY: LOW] {Reading the entire `global.jsonl` into memory is treated as harmless. That assumption holds only while the file is small. As history grows, a one-shot full rewrite becomes the exact kind of maintenance task that fails unexpectedly in production because nobody budgeted for log size.}

## Verdict

BLOCKED_HIGH_SEVERITY
