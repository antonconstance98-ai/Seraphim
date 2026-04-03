# Phase 06 — Codex Plan Review

**Reviewed:** 2026-04-03T02:35:33.077Z
**Model:** gpt-5.4
**Plans reviewed:** 06-01-PLAN.md, 06-02-PLAN.md
**Review type:** Multi-round (constructive + adversarial)

## Findings

=== Round 1 (constructive) ===
[PLAN 01] [SEVERITY: HIGH] The task spec is incomplete for execution as shown: the `htmlEscape(str)` action is truncated (`then escapes ...`) and the rest of the implementation steps are missing. An executor would need the full escaping contract and any remaining required functions/behavior before coding safely.

[PLAN 01] [SEVERITY: MEDIUM] No explicit verification commands are provided. The plan needs concrete checks such as loading sample records from `~/.claude/dashboard/global.jsonl`, exercising `computeMetrics([])`, exercising malformed numeric fields through `safeNum`, and calling `ensureChartJs()` in both cached/offline paths.

[PLAN 01] [SEVERITY: MEDIUM] Requirements coverage is unclear for `INTG-05` and likely incomplete in practice: the plan creates `~/.claude/hooks/codex-dashboard-generator.js`, but it does not include any task to wire or validate integration in `~/.claude/hooks/codex-global-aggregator.js`. If integration is already expected to exist, the plan should say that explicitly and include a verification command proving it.

[PLAN 01] [SEVERITY: LOW] The standalone behavior is referenced but not specified. The header says `node codex-dashboard-generator.js (TTY detection)`, but there is no concrete task action or verification describing CLI entrypoint behavior, exit conditions, or output when run directly.

[PLAN 01] [SEVERITY: LOW] No dependency issue is visible in the provided text, but the plan should explicitly state that it does not depend on Plan 02 artifacts and must not reference HTML rendering code from the later wave. That boundary is implied in the objective, not enforced in the task steps.

[PLAN 01] [SEVERITY: LOW] No parallel file conflict can be validated from the single plan provided. If other wave-1 plans also touch `~/.claude/hooks/codex-dashboard-generator.js` or `~/.claude/dashboard/assets/chart.min.js`, that conflict is not disclosed here and should be checked.

---

=== Round 2 (adversarial) ===
I’m reviewing the plan as an adversarial critic and checking where its assumptions, edge cases, and production failure modes break down.
[CONCERN] [SEVERITY: HIGH] {The plan assumes `ensureChartJs()` can “download if online, return false if offline” without specifying any timeout, HTTP error handling contract, integrity check, or partial-write protection. The simplest production failure is a hanging or truncated download that leaves a corrupt `chart.min.js` in place while still satisfying the file-presence expectation.}

[CONCERN] [SEVERITY: HIGH] {It treats Chart.js caching as operationally harmless, but pins a CDN URL with no checksum verification. A bad CDN response, captive portal HTML, proxy injection, or partial response could be cached as `chart.min.js` and silently poison later dashboard renders. Technically the sidecar exists; practically the dashboard is broken.}

[CONCERN] [SEVERITY: HIGH] {The plan relies on `computeMetrics(records)` accepting “parsed record objects” but does not define how malformed object shape is handled beyond a few numeric coercions. If `tokens` is missing, non-object, or partially missing, the model split and cache-efficiency math can throw or produce `NaN` unless every nested access is guarded. The schema examples are not a contract.}

[CONCERN] [SEVERITY: HIGH] {“Records missing timestamp are excluded” is underspecified. Missing from what exactly: all aggregates, block log, project table, model split, task distribution? If exclusion is applied inconsistently, totals diverge across sections and the dashboard becomes internally contradictory while still looking valid.}

[CONCERN] [SEVERITY: HIGH] {The requirement “computeMetrics([]) returns a zeroed structure with empty arrays and objects, never throws” does not cover non-array inputs. If `generateDashboard` passes `null`, a scalar, or a parse-corrupted value, the implementation can still fail in production despite technically satisfying the empty-array requirement.}

[CONCERN] [SEVERITY: MEDIUM] {The plan hardcodes only `gpt-5.4` and `gpt-5.4-mini` in `modelSplit` even though upstream schema evolution could add other models. That means real cost and token usage can disappear from model-specific reporting while still being counted elsewhere, making the dashboard technically complete at top level but practically misleading.}

[CONCERN] [SEVERITY: MEDIUM] {Alphabetical sorting for `projectTable` plus a special “Unattributed row last” rule assumes project names are stable, normalized, and consistently cased. Variants like `app`, `App`, trailing whitespace, or empty-string names can create duplicate-looking projects or unstable ordering.}

[CONCERN] [SEVERITY: MEDIUM] {The plan assumes `project_name` is the right grouping key but the source data also has `project_path`. If `project_name` collides across different paths, costs and calls from unrelated projects merge into one row. That is the simplest way to produce a plausible but false dashboard in production.}

[CONCERN] [SEVERITY: MEDIUM] {`cacheEfficiency` uses `(sum_cached_input / (sum_input + sum_cached_input) * 100).toFixed(1)` without stating how negative values, absurdly large values, or inconsistent token accounting are handled. Upstream bad data can produce percentages that are mathematically valid but operationally nonsensical.}

[CONCERN] [SEVERITY: MEDIUM] {The block log rule filters on `verdict==='BLOCK' AND block_summary is a non-empty string`, but “non-empty” is not defined. Whitespace-only summaries, non-string summaries, or summaries containing formatting garbage can slip through or be dropped inconsistently.}

[CONCERN] [SEVERITY: MEDIUM] {Session history excludes `null session_id` entirely, which satisfies the requirement but may hide a material fraction of usage. The plan treats this as acceptable despite the context explicitly saying 7 live records already have null session IDs. Technically compliant, practically underreporting.}

[CONCERN] [SEVERITY: MEDIUM] {The plan assumes JSONL parsing in `generateDashboard` is routine, but JSONL files fail in mundane ways: blank lines, partial trailing writes, BOMs, and single-line corruption. If one bad line aborts the whole read path, the dashboard generator becomes fragile to normal production logging behavior.}

[CONCERN] [SEVERITY: MEDIUM] {The requirement that `computeMetrics` is a pure transform conflicts with the broader plan’s dependence on live file IO in `generateDashboard`, yet there is no clear boundary for parse failures vs metric failures. That over-simplifies debugging: a broken dashboard can come from IO, parsing, or aggregation, but the plan collapses them into one flow.}

[CONCERN] [SEVERITY: LOW] {The plan assumes `savingsPct` and `catchRate` being strings is harmless, but stringified percentages often sort lexically, not numerically, in later render layers. This is a common “technically satisfied, practically broken” trap once Plan 02 introduces HTML tables and charts.}

[CONCERN] [SEVERITY: LOW] {“Returns a DASHBOARD_DATA object with 7 top-level keys” is a shape-level requirement, not a correctness requirement. An implementation can satisfy it while populating sections with internally inconsistent totals, empty placeholders, or silently dropped categories. The plan over-indexes on structure and under-specifies invariants.}

[CONCERN] [SEVERITY: LOW] {The live-data context says only `gpt-5.4` is currently observed, but the plan treats future `gpt-5.4-mini` support as sufficient forward-compatibility. That is an unjustified assumption; the next real production surprise is more likely an unplanned model or malformed record than the one hypothetical model they named.}

## Verdict

BLOCKED_HIGH_SEVERITY
