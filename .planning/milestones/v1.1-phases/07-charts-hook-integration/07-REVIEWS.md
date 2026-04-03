# Phase 07 — Codex Plan Review

**Reviewed:** 2026-04-03T04:28:05.494Z
**Model:** gpt-5.4
**Plans reviewed:** 07-01-PLAN.md, 07-02-PLAN.md
**Review type:** Multi-round (constructive + adversarial)

## Findings

=== Round 1 (constructive) ===
[PLAN 01] [SEVERITY: HIGH] CHART-03 is not actually verified: Task 2 only checks that `typeof Chart` text exists in generated HTML, and Task 3 opens the normal dashboard with Chart.js inlined, so there is no command that proves the page exits cleanly when `Chart` is absent.

[PLAN 01] [SEVERITY: MEDIUM] Task 2 verification is inconsistent with the stated implementation. The action says to guard with `if(!series||!series.length)return;`, but the grep-based check requires the literal string `series.length===0`, so an implementation that matches the task can still fail verification.

[PLAN 01] [SEVERITY: LOW] The plan output requires creating `.planning/phases/07-charts-hook-integration/07-01-SUMMARY.md`, but that file is not listed in `files_modified`, leaving the declared write set incomplete.

[PLAN 02] [SEVERITY: MEDIUM] Dependency handling is underspecified. The task requires `/home/alucard/.claude/hooks/codex-global-aggregator.js` and checks it for `generateDashboard`, but `depends_on` is empty and the plan does not say whether execution should stop or continue if that prerequisite is missing.

[PLAN 02] [SEVERITY: MEDIUM] INTG-02 says the change must be idempotent, but the verification only proves that one matching hook exists after `codex-cost-reporter`; it does not fail on duplicate `codex-global-aggregator.js` entries from repeated runs.

[PLAN 02] [SEVERITY: LOW] The plan output requires creating `.planning/phases/07-charts-hook-integration/07-02-SUMMARY.md`, but that file is not listed in `files_modified`, so the declared file set is incomplete.

---

=== Round 2 (adversarial) ===
[CONCERN] [SEVERITY: HIGH] {`07-01` assumes `require('/home/alucard/.claude/hooks/codex-dashboard-generator')` is safe in verification, but hook-style scripts often execute side effects at module load. If this file generates output, reads missing files, or exits on import, the verification path is invalid and the plan fails before testing the actual feature.}

[CONCERN] [SEVERITY: HIGH] {The date-validation logic is overconfident. `new Date(ts)` plus `toISOString().slice(0,10)===ts.slice(0,10)` rejects some bad inputs, but it still relies on JavaScript date parsing semantics for non-canonical ISO strings. That behavior is implementation-sensitive enough that “normalized dates rejected” is not actually guaranteed by the plan.}

[CONCERN] [SEVERITY: HIGH] {Weekly regrouping is underspecified and likely wrong in practice. “ISO Monday-UTC via Thursday algorithm” sounds precise, but the plan never defines the label format, year rollover handling, or what happens for partial weeks. This is the simplest production failure: the toggle appears to work while silently mis-bucketing end-of-year data.}

[CONCERN] [SEVERITY: HIGH] {The plan claims CHART-01 is satisfied, but the line chart data source is `DASHBOARD_DATA.timeSeries` while the bar chart uses `DASHBOARD_DATA.projectTable`. That means one chart is time-based and the other is project-based. If CHART-01 expects cost/baseline and savings trends in the same temporal context, the requirement may be technically “rendered” but analytically broken.}

[CONCERN] [SEVERITY: HIGH] {“Chart.js inlined from sidecar; no external script src URLs” assumes the sidecar file is present and readable at generation time. The plan only guards for `typeof Chart==='undefined'` in the browser, not for missing `chart.min.js` during HTML generation. If the sidecar is absent, generator execution likely crashes before the dashboard exists.}

[CONCERN] [SEVERITY: MEDIUM] {The plan treats “silent return when Chart is absent” as success, but that can leave a dashboard with empty chart regions and no indication anything failed. That technically avoids a crash while practically shipping a broken UI with zero observability.}

[CONCERN] [SEVERITY: MEDIUM] {The verification for `07-01` is mostly grep-driven. Searching for `costSavingsChart`, `groupTS`, `setChartGrouping`, and the absence of `<script src=` proves strings exist in HTML, not that the JavaScript is syntactically valid, runs in the right order, binds to actual DOM nodes, or produces correct charts.}

[CONCERN] [SEVERITY: MEDIUM] {`buildLine(mode): destroy+null inst; if(!series||!series.length)return` assumes a chart instance variable exists in a stable closure and is not leaked across rerenders. The plan never states where the instance lives or how repeated calls from the toggle avoid referencing an uninitialized or shadowed variable.}

[CONCERN] [SEVERITY: MEDIUM] {The bar chart requirement is described as “savings% render in dashboard,” but sorting `parseFloat(savingsPct)` assumes `savingsPct` is a parseable numeric string already stripped of `%`, commas, placeholders, or empty values. One formatted value like `"12.4%"` or `"N/A"` will either sort incorrectly or collapse to `NaN`.}

[CONCERN] [SEVERITY: MEDIUM] {Filtering out `r.name!=='Unattributed'` hardcodes presentation logic into chart generation and assumes exact casing and spelling. If the upstream data uses `unattributed`, `Unattributed `, or a localized label, the supposedly excluded bucket leaks into production charts.}

[CONCERN] [SEVERITY: MEDIUM] {Canvas heights `h=100` and `h=120` are arbitrary and unvalidated. The plan never addresses container sizing, responsive layout, long project-name labels, or density limits. The simplest real-world failure is charts technically rendering but becoming unreadable with even modest data volume.}

[CONCERN] [SEVERITY: MEDIUM] {The human verification step says “open file://... confirm charts render, Weekly toggle works,” which is weak to the point of being non-verification. It ignores empty datasets, single-point datasets, malformed savings values, high-cardinality project tables, and browser-specific behavior.}

[CONCERN] [SEVERITY: MEDIUM] {`07-02` assumes `s.hooks.SessionStart.find(...)` matches the intended hook group uniquely. If there are multiple `SessionStart` groups containing `codex-cost-reporter`, the plan silently modifies the first one and declares success while leaving actual runtime behavior ambiguous.}

[CONCERN] [SEVERITY: MEDIUM] {Idempotency is asserted, not demonstrated. The plan checks whether `"codex-global-aggregator"` is already in “that group,” but if the same hook exists elsewhere in `SessionStart`, repeated executions can still leave duplicate effective runs across groups.}

[CONCERN] [SEVERITY: MEDIUM] {“All other sections unchanged” is fragile if the edit path reparses and rewrites JSON. Key ordering, whitespace, comments if present, or formatting conventions may change even when semantic content does not. The plan verifies validity, not preservation.}

[CONCERN] [SEVERITY: LOW] {The precheck `grep -c generateDashboard ... >=1` is a bad proxy for hook readiness. A file can contain that string and still be non-executable, invalid JavaScript, or point at the wrong runtime behavior.}

[CONCERN] [SEVERITY: LOW] {The `timeout:30` requirement is treated as sufficient without any assumption check on actual runtime duration. If the aggregator grows slower over time, the hook can start timing out in production even though the plan permanently codifies 30 seconds as acceptable.}

[CONCERN] [SEVERITY: LOW] {Both plans assume the absolute paths under `/home/alucard/.claude/...` are stable deployment targets. That is technically satisfied in this environment but practically broken for portability, alternate users, cloned setups, or test environments.}

## Verdict

BLOCKED_HIGH_SEVERITY
