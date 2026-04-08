# Phase 14: Three-Model Reporting - Research

**Researched:** 2026-04-03
**Domain:** Node.js reporting pipeline, Chart.js, JSONL data transformation
**Confidence:** HIGH

## Summary

This phase updates four existing hook files to recognize MiniMax as a third model in the cost tracking and dashboard pipeline. The infrastructure is already solid: `codex-pricing.js` has MiniMax pricing since Phase 8, `global.jsonl` already captures any model that logs a `[CODEX_RESULT]` marker (modelSplit in `computeMetrics` dynamically creates entries for unknown models), and the aggregator discovers all records from the same JSONL files regardless of model.

The scope is incremental: add MiniMax pass-through to `codex-token-logger.js` for v2.0 log fields, extend `codex-cost-reporter.js` to surface per-model breakdowns and fallback counts, add the MiniMax series to charts in `codex-dashboard-generator.js`, and add a Fallback Events panel as a new dashboard section. No structural changes are needed to `codex-global-aggregator.js` — MiniMax entries already flow through it unchanged.

The main technical work is in the dashboard generator: adding a third dataset to the Chart.js line chart, updating the pie/cost-breakdown visualization, and building the Fallback Events table from records that have `source: 'api-fallback'` and `model: 'minimax-m2.7'` (emitted by `codex-handoff.js` Phase 13).

**Primary recommendation:** Work file-by-file — token logger first (v2.0 field pass-through), then cost reporter (three-model breakdown + fallback count), then dashboard generator (MiniMax series + Fallback Events panel). The aggregator needs zero changes.

---

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions

**Dashboard layout:**
- D-01: Combined charts with a third color/series for MiniMax. Existing time-series chart gets a MiniMax line. Cost breakdown pie chart shows Opus vs Codex vs MiniMax split.
- D-02: New "Fallback Events" panel showing when Codex→MiniMax fallbacks occurred, why (rate limit, quota, timeout), and how much each fallback cost. This is a new dashboard section, not a modification to existing charts.
- D-03: Savings calculation uses corrected Opus 4.6 baseline ($5/$25, not the old $15/$75). All savings percentages reflect the true Opus 4.6 cost.

**Token logging updates:**
- D-04: `codex-token-logger.js` already handles different models via the `model` field. Ensure it correctly recognizes `minimax-m2.7` entries (from Phase 11 bug scanner, Phase 9 dual review, Phase 10 adversarial review, Phase 12 compression, Phase 13 execution fallback).
- D-05: New log fields for v2.0 entries: `dual_review: true/false` (Phase 9), `review_round: 1|2` + `round_model` (Phase 10), `fallback_from: 'codex'` (Phase 13), `compression: true` (Phase 12). Backward compatible — old entries without these fields still parse fine.

**Cost reporter updates:**
- D-06: `codex-cost-reporter.js` SessionStart report shows three-model breakdown: Opus orchestration cost, Codex execution cost, MiniMax analysis cost, total, and savings vs Opus-only baseline.
- D-07: Report includes fallback event count and cost: "Codex→MiniMax fallbacks: N events, $X.XX additional cost."

**Global aggregator updates:**
- D-08: `codex-global-aggregator.js` already aggregates token-log.jsonl across projects. No discovery changes needed — MiniMax entries are in the same JSONL files. Pricing computation updated via the corrected `codex-pricing.js` from Phase 8.

**Dashboard generator updates:**
- D-09: `codex-dashboard-generator.js` adds MiniMax as third series in Chart.js charts. Color scheme: Opus = existing color, Codex = existing color, MiniMax = new distinct color (e.g., teal or green — Claude's discretion).
- D-10: Fallback events panel rendered as a table: date, reason, tokens consumed, cost, source project. Sorted by most recent first.

### Claude's Discretion
- Exact color for MiniMax in charts
- Fallback events panel position on dashboard (top, bottom, sidebar)
- Whether to add a "Model Efficiency" metric (quality per dollar)
- Summary statistics format in SessionStart report

### Deferred Ideas (OUT OF SCOPE)
- Per-model quality tracking (pass rate, retry rate) alongside cost metrics — requires structured verdict logging beyond current ALLOW/BLOCK
- Export dashboard data as JSON API for external consumption
- Email/Slack notification when daily spend approaches $15 limit
</user_constraints>

---

## Standard Stack

### Core (all already installed)
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| Node.js | v22.22.0 | All hook scripts | Project-established runtime — all existing hooks are Node.js |
| Chart.js | 4.5.1 | Dashboard charts | Already pinned in dashboard generator with SHA-256 integrity check |
| fs (built-in) | Node.js built-in | Atomic file I/O | Pattern established: write-to-temp + renameSync |
| path (built-in) | Node.js built-in | Path resolution | Used by all existing hooks |

### No New Dependencies
All libraries needed for Phase 14 are already present. No `npm install` required.

**Key modules to modify (all at `~/.claude/hooks/`):**
- `codex-token-logger.js` — PostToolUse hook, logs [CODEX_RESULT] markers
- `codex-cost-reporter.js` — SessionStart hook, Markdown report generator
- `codex-global-aggregator.js` — SessionStart hook, JSONL merge pipeline (READ ONLY for this phase — no functional changes)
- `codex-dashboard-generator.js` — HTML dashboard generator (most work)
- `codex-pricing.js` — Pricing module (already has MiniMax, no changes needed)

## Architecture Patterns

### Existing Data Flow (unchanged)
```
PostToolUse → codex-token-logger.js
              writes {cwd}/.planning/token-log.jsonl

SessionStart → codex-cost-reporter.js
               reads {cwd}/.planning/token-log.jsonl
               writes {cwd}/.planning/session-reports/YYYY-MM-DD.md

SessionStart → codex-global-aggregator.js
               discovers all token-log.jsonl across ~/projects, ~/agent, etc.
               merges into ~/.claude/dashboard/global.jsonl (dedup by session_id|timestamp)
               calls generateDashboard(DASHBOARD_DIR)
               ↓
               codex-dashboard-generator.js
               reads ~/.claude/dashboard/global.jsonl
               writes ~/.claude/dashboard/dashboard.html (atomic)
```

### Pattern 1: [CODEX_RESULT] Marker (codex-token-logger.js)
**What:** `codex-token-logger.js` only logs records when `tool_result` contains `[CODEX_RESULT]` as a JSON-prefixed marker. It exits silently for all other PostToolUse events.

**Critical insight for D-04/D-05:** The logger currently passes `codexResult` fields through to the log record. However it does NOT forward arbitrary additional fields — only the explicitly-listed ones (`model`, `source`, `task_type`, `tokens`, `cost_usd`, `rate_limit_pct`). To capture v2.0 fields (`dual_review`, `review_round`, `round_model`, `fallback_from`, `compression`), the logger must be extended to read and conditionally include these optional fields from the parsed codexResult payload.

```javascript
// Current record construction (line 62-76 of codex-token-logger.js):
const record = {
  timestamp:      new Date().toISOString(),
  session_id:     data.session_id || null,
  model:          codexResult.model,
  source:         codexResult.source || 'cli',
  task_type:      codexResult.task_type || 'unknown',
  tokens: { ... },
  cost_usd:       computeCost(codexResult.tokens, codexResult.model),
  rate_limit_pct: codexResult.rate_limit_pct || null
};
// D-05: v2.0 fields to add conditionally (backward compatible):
// if (codexResult.dual_review !== undefined) record.dual_review = codexResult.dual_review;
// if (codexResult.review_round !== undefined) record.review_round = codexResult.review_round;
// if (codexResult.round_model !== undefined) record.round_model = codexResult.round_model;
// if (codexResult.fallback_from !== undefined) record.fallback_from = codexResult.fallback_from;
// if (codexResult.compression !== undefined) record.compression = codexResult.compression;
```

**Backward compatibility:** Old entries without these fields parse fine since all consumers use optional chaining (`rec.fallback_from || null`).

### Pattern 2: MiniMax Fallback Detection (for Fallback Events panel)
**What:** Fallback events are records where `source === 'api-fallback'` AND `model === 'minimax-m2.7'`. These are emitted by `codex-handoff.js` (Phase 13) when Codex is rate-limited.

**Fallback reason detection:** `codex-handoff.js` does NOT currently emit a `fallback_reason` field. The `[CODEX_RESULT]` marker it emits contains only `model`, `source`, `task_type`, `tokens`, `rate_limit_pct`. The reason (rate limit/quota/timeout) can be inferred from `codexResult.error` in the calling scope, but this information is not currently propagated to the log.

**Implication for D-10:** The "reason" column in the Fallback Events table will have limited precision. Options:
1. Use `rate_limit_pct` field (already logged): if `rate_limit_pct >= 95`, reason = "Rate limit". Otherwise label as "Unknown (Codex failed)".
2. Add a `fallback_reason` field to the `[CODEX_RESULT]` marker in `codex-handoff.js` (requires touching Phase 13 code).

**Recommendation:** Option 1 — infer from existing fields. No changes to `codex-handoff.js`. The Fallback Events panel shows date, project, tokens, cost, and source model. "Reason" column shows "Rate limit" when `rate_limit_pct >= 95`, "Codex failure" otherwise. This is sufficient for the health indicator goal stated in D-02.

### Pattern 3: Three-Model Cost Breakdown (cost-reporter.js)
**What:** `generateReport()` already groups records by model in `byModel`. Adding MiniMax is automatic — the loop handles any model string. The change is:
1. Pre-initializing MiniMax with zeros (so it appears even with no calls, like gpt-5.4-mini)
2. Adding three-model summary section to the Markdown report
3. Adding fallback event count (filter records where `source === 'api-fallback'` AND `model === 'minimax-m2.7'`)

**Corrected baseline (D-03):** `codex-pricing.js` already has `OPUS_PRICING = { input: 5.00, cached_input: 1.25, output: 25.00 }` (Opus 4.6 corrected pricing from Phase 8). The `computeOpusCost()` function uses this. No changes needed to baseline computation.

```javascript
// Fallback event detection in generateReport():
const fallbackEvents = records.filter(r =>
  r.source === 'api-fallback' && r.model === 'minimax-m2.7'
);
const fallbackCost = fallbackEvents.reduce((sum, r) =>
  sum + (typeof r.cost_usd === 'number' ? r.cost_usd : 0), 0
);
```

### Pattern 4: Chart.js Third Series (dashboard-generator.js)
**What:** The existing line chart (`buildLine()`) has two datasets: "Actual Cost ($)" and "Opus Baseline ($)". Adding MiniMax requires:
1. Accumulating `minimax_cost` alongside `cost` and `opusBaseline` in the `byDate` Map
2. Adding it as a third dataset in `buildLine()`

**timeSeries data shape change:**
```javascript
// Current: { date, cost, opusBaseline }
// Updated: { date, cost, opusBaseline, minimaxCost }
// 'cost' = total of all non-Opus models (Codex + MiniMax combined) — preserves existing semantics
// 'minimaxCost' = MiniMax-only cost (for the third series)
```

**Model colors (Claude's discretion):**
- Opus Baseline: `#ffab00` (existing — amber/warning)
- Actual Cost (Codex): `#00d4ff` (existing — cyan/accent)
- MiniMax: `#00e676` (green — `--text-success` already in CSS variables)

**Chart.js 4.5.1 multi-dataset pattern:**
```javascript
datasets: [
  { label: 'Actual Cost ($)', data: ..., borderColor: '#00d4ff', ... },
  { label: 'Opus Baseline ($)', data: ..., borderColor: '#ffab00', ... },
  { label: 'MiniMax Cost ($)', data: ..., borderColor: '#00e676',
    backgroundColor: 'rgba(0,230,118,0.08)', fill: false, tension: 0.3, pointRadius: 3 }
]
```

### Pattern 5: modelSplit Pre-initialization (dashboard-generator.js)
**What:** In `computeMetrics()`, `modelSplit` currently pre-initializes only `gpt-5.4` and `gpt-5.4-mini` with zeros (lines 336-347). To show MiniMax even with no calls, add `'minimax-m2.7'` to the initialization block.

```javascript
// Current pre-initialization (lines 336-347):
const modelSplit = {
  'gpt-5.4': { calls: 0, tokens: {...}, cost: 0 },
  'gpt-5.4-mini': { calls: 0, tokens: {...}, cost: 0 }
};
// Add:
// 'minimax-m2.7': { calls: 0, tokens: { input: 0, cached_input: 0, output: 0 }, cost: 0 }
```

### Pattern 6: Fallback Events Panel (dashboard-generator.js)
**What:** New HTML section. Similar structure to the existing "BLOCK Log" section (lines 902-906 of dashboard-generator.js). Implemented as a server-side rendered table (not client-side JS) for simplicity — consistent with blockLogHtml pattern.

**Data source:** Records from `global.jsonl` where `source === 'api-fallback'` AND `model === 'minimax-m2.7'`. Collect in `computeMetrics()` alongside `blockLogEntries`.

**Panel position (Claude's discretion):** Place after "Model Split" section and before "Task Type Distribution". Rationale: fallback events are a model-routing health indicator, most relevant immediately after seeing the model breakdown.

**Table columns:** Date, Project, Task Type, Tokens Used, Cost, Rate Limit %.
Sorted by most recent first (same as blockLog).

```javascript
// Data collection in computeMetrics() single pass:
const fallbackEntries = [];
// Inside the main loop:
if (r.source === 'api-fallback' && r.model === 'minimax-m2.7') {
  fallbackEntries.push({
    timestamp:    r.timestamp,
    projectName:  r.project_name,
    taskType:     r.task_type || 'unknown',
    tokens:       (r.tokens && r.tokens.input || 0) + (r.tokens && r.tokens.output || 0),
    cost:         safeNum(r.cost_usd),
    rateLimitPct: r.rate_limit_pct || null
  });
}
// Sort desc by timestamp after loop:
const fallbackLog = fallbackEntries.sort((a, b) =>
  a.timestamp > b.timestamp ? -1 : a.timestamp < b.timestamp ? 1 : 0
);
```

### Anti-Patterns to Avoid
- **Touching `codex-global-aggregator.js`:** Zero changes needed. MiniMax entries flow through identically to Codex entries. Touching it risks breaking the mtime cache or dedup logic.
- **Touching `codex-pricing.js`:** MiniMax pricing was added in Phase 8. Already correct. No changes.
- **Changing the JSONL schema in `global.jsonl`:** The aggregator enriches records with `project_path`, `project_name`, `opus_baseline_usd` at merge time. MiniMax records have the same schema as Codex records. No migration needed.
- **Adding a `pie` chart by importing additional Chart.js plugins:** Chart.js 4.5.1 UMD bundle already includes the pie/doughnut chart type. No additional download or SHA update needed for a basic pie chart.
- **Regenerating the SHA-256 for Chart.js:** The pinned hash `48444a82...` is for Chart.js 4.5.1 and remains valid. Do not change it.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Atomic dashboard writes | Custom locking mechanism | `write to .tmp + fs.renameSync()` (already in codebase) | Atomic on Linux; pattern already established in Phase 6 |
| Cost computation | Inline pricing math | `computeCost()` / `computeOpusCost()` from `codex-pricing.js` | All pricing centralized; corrections apply retroactively |
| HTML escaping | Custom sanitizer | `htmlEscape()` already in dashboard-generator.js | Already handles null/numbers/undefined |
| JSONL dedup | Timestamp-only comparison | `buildDedupKey(rec)` = `session_id|timestamp` | Handles null session_id correctly (Phase 5 decision) |
| Chart.js download | Custom CDN fetch | `ensureChartJs()` already in dashboard-generator.js | SHA-256 integrity verified; handles offline gracefully |

**Key insight:** Every utility needed for Phase 14 already exists in the codebase. This phase is purely additive — extend existing data structures and rendering functions, not replace them.

## Common Pitfalls

### Pitfall 1: token-log.jsonl vs global.jsonl Token Field Names
**What goes wrong:** `token-log.jsonl` uses `{ input, cached_input, output }`. The `[CODEX_RESULT]` marker payload uses `{ input_tokens, cached_input_tokens, output_tokens }`. The `computeOpusCost()` function expects the `{ input, cached_input, output }` shape.
**Why it happens:** Two schemas exist because `codex-exec.js` normalizes from the Codex CLI JSONL event schema to the log schema on write.
**How to avoid:** Never pass raw `codexResult.tokens` (which has `input_tokens` shape) to `computeOpusCost()`. Only pass `record.tokens` (from parsed JSONL, which has `input` shape). The logger performs this normalization at lines 68-73.
**Warning signs:** `computeOpusCost()` returning 0 or incorrect values for non-zero token counts.

### Pitfall 2: modelSplit Dynamic vs Pre-initialized Keys
**What goes wrong:** If MiniMax is not pre-initialized in `modelSplit`, the "Model Split" table on the dashboard shows no MiniMax row during sessions where no MiniMax calls occurred. This is confusing when users are debugging why MiniMax isn't being used.
**Why it happens:** `modelSplit` only grows from observed data in the current single-pass loop.
**How to avoid:** Pre-initialize `'minimax-m2.7'` alongside `'gpt-5.4'` and `'gpt-5.4-mini'` at the start of `computeMetrics()`. This matches the existing pattern for the two Codex models.

### Pitfall 3: timeSeries minimaxCost Accumulation
**What goes wrong:** If `minimaxCost` is accumulated only from records with `model === 'minimax-m2.7'`, but the existing `cost` field accumulates ALL models (including MiniMax), the total line chart shows double-counting for MiniMax when all three series are shown.
**Why it happens:** The existing `cost` field represents total actual spend across all models.
**How to avoid:** Keep `cost` as the total (Codex + MiniMax combined). The MiniMax series shows MiniMax-only spend as an additional breakdown. In the chart legend, label it "MiniMax Cost ($)" not "MiniMax + Actual". The line for "Actual Cost ($)" already subsumes MiniMax.
**Alternative:** Show three separate lines: Codex-only, MiniMax-only, Opus Baseline. This is cleaner but requires tracking `codexCost` separately too. CONTEXT.md D-01 says "Existing time-series chart gets a MiniMax line" — implying addition of one new line, not a restructure of the existing two.

### Pitfall 4: Fallback Events Panel — Empty State
**What goes wrong:** If `fallbackLog` is empty (no MiniMax fallbacks yet), the panel renders nothing or crashes.
**Why it happens:** No MiniMax fallbacks exist in the current global.jsonl (all 222 records are `gpt-5.4`). The panel will be empty on first render.
**How to avoid:** Use the same empty-state pattern as `blockLogHtml`:
```javascript
const fallbackLogHtml = fallbackLog.length === 0
  ? '<p style="color: var(--text-secondary); font-style: italic;">No Codex→MiniMax fallback events recorded.</p>'
  : fallbackLog.map(entry => { ... }).join('');
```

### Pitfall 5: Cost Reporter — Opus Model Not in byModel
**What goes wrong:** Opus (Claude) is the orchestrator but its costs are never logged to `token-log.jsonl` — those records live in `~/.claude/projects/.../session.jsonl` not the Codex pipeline. The "Opus orchestration cost" in D-06 is the *baseline* (what it would have cost to run everything on Opus), not actual Opus spend.
**Why it happens:** The project is designed so Opus only runs as orchestrator (planning, directing) and its token usage is in Claude Code's own session files, not the Codex token log.
**How to avoid:** The SessionStart report's "three-model breakdown" should be: Codex cost (sum of records with model `gpt-5.4` or `gpt-5.4-mini`), MiniMax cost (sum of `minimax-m2.7`), Opus baseline (computed from `computeOpusCost()` on all records). Label clearly: "Opus Baseline (what this would have cost)" not "Opus Actual Cost".

### Pitfall 6: [CODEX_RESULT] Marker Emission for v2.0 Fields
**What goes wrong:** The v2.0 optional fields (`dual_review`, `fallback_from`, etc.) are emitted in the `[CODEX_RESULT]` marker by Phase 9/10/12/13 hooks. But `codex-token-logger.js` currently only reads explicitly listed fields and ignores anything else. The fields exist in the marker payload but are silently discarded.
**Why it happens:** The logger was written before v2.0 fields were defined.
**How to avoid:** Add conditional inclusion of v2.0 fields in the record construction:
```javascript
if (codexResult.dual_review !== undefined)  record.dual_review  = codexResult.dual_review;
if (codexResult.review_round !== undefined) record.review_round = codexResult.review_round;
if (codexResult.round_model !== undefined)  record.round_model  = codexResult.round_model;
if (codexResult.fallback_from !== undefined) record.fallback_from = codexResult.fallback_from;
if (codexResult.compression !== undefined)  record.compression  = codexResult.compression;
```

## Code Examples

Verified patterns from the existing codebase:

### Existing: [CODEX_RESULT] marker emission (codex-handoff.js lines 155-161)
```javascript
console.error(`[CODEX_RESULT] ${JSON.stringify({
  model:          'gpt-5.4',
  source:         'codex-cli',
  task_type:      'execution',
  tokens:         normalizedTokens,
  rate_limit_pct: codexResult.rateLimitPct !== undefined ? codexResult.rateLimitPct : null,
})}`);
```

### Existing: MiniMax model fallback marker (codex-handoff.js lines 231-237)
```javascript
console.error(`[CODEX_RESULT] ${JSON.stringify({
  model:          'minimax-m2.7',
  source:         'api-fallback',
  task_type:      'execution',
  tokens:         normalizedTokens,
  rate_limit_pct: null,
})}`);
```

### Existing: modelSplit pre-initialization pattern (dashboard-generator.js lines 336-347)
```javascript
const modelSplit = {
  'gpt-5.4': {
    calls: 0,
    tokens: { input: 0, cached_input: 0, output: 0 },
    cost: 0
  },
  'gpt-5.4-mini': {
    calls: 0,
    tokens: { input: 0, cached_input: 0, output: 0 },
    cost: 0
  }
};
// Phase 14: add 'minimax-m2.7' with same structure
```

### Existing: blockLog collection pattern (dashboard-generator.js)
```javascript
// Collection in single-pass loop:
if (r.verdict === 'BLOCK' && typeof r.block_summary === 'string' && r.block_summary.trim().length > 0) {
  blockLogEntries.push({ timestamp, projectName, taskType, reviewTaskType, summary });
}
// After loop:
const blockLog = blockLogEntries.sort((a, b) =>
  a.timestamp > b.timestamp ? -1 : a.timestamp < b.timestamp ? 1 : 0
);
// HTML empty state:
const blockLogHtml = data.blockLog.length === 0
  ? '<p style="...">No BLOCK events recorded.</p>'
  : data.blockLog.map(entry => `<div class="block-log-item">...</div>`).join('');
```

### Existing: atomic write pattern (dashboard-generator.js lines 1143-1147)
```javascript
fs.mkdirSync(dir, { recursive: true });
const tmpPath = path.join(dir, '.dashboard.html.tmp.' + process.pid);
fs.writeFileSync(tmpPath, html, 'utf8');
fs.renameSync(tmpPath, path.join(dir, 'dashboard.html'));
```

### Existing: Chart.js multi-dataset line chart (dashboard-generator.js lines 1039-1080)
```javascript
lineInst = new Chart(ctx, {
  type: 'line',
  data: {
    labels: series.map(p => p.date),
    datasets: [
      { label: 'Actual Cost ($)', data: series.map(p => p.cost),
        borderColor: '#00d4ff', backgroundColor: 'rgba(0,212,255,0.08)',
        fill: false, tension: 0.3, pointRadius: 3 },
      { label: 'Opus Baseline ($)', data: series.map(p => p.opusBaseline),
        borderColor: '#ffab00', backgroundColor: 'rgba(255,171,0,0.08)',
        fill: false, tension: 0.3, pointRadius: 3 }
      // Phase 14: add third dataset for MiniMax
    ]
  },
  options: { ... }  // unchanged
});
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Opus-only cost tracking | Two-model (Codex + Opus baseline) | v1.0 Phase 4 | This phase extends to three models |
| Inline OPUS_PRICING in cost-reporter | Centralized `codex-pricing.js` | v1.1 Phase 5 | Phase 14 benefits: pricing corrections are automatic |
| $15/$75 Opus pricing | $5/$25 Opus 4.6 pricing | Phase 8 | Phase 14 uses corrected baseline throughout |
| blockLog (review events) | blockLog + fallbackLog (new in Phase 14) | Phase 14 | New health indicator panel |

**No deprecated patterns in this phase** — all existing code follows current conventions.

## Environment Availability

Step 2.6: SKIPPED for most dependencies — this phase is entirely code/config changes to existing hook files with no new external dependencies.

| Dependency | Required By | Available | Version | Fallback |
|------------|------------|-----------|---------|----------|
| Node.js | All hook scripts | Yes | v22.22.0 | — |
| Chart.js 4.5.1 | Dashboard charts | Yes (cached) | 4.5.1 | SVG fallback (existing) |
| `codex-pricing.js` | Cost computation | Yes | Phase 8 version | — |
| `global.jsonl` | Dashboard data | Yes | 222 records (gpt-5.4 only) | — |

**Note on live MiniMax data:** `global.jsonl` currently has 222 records all with `model: 'gpt-5.4'`. No MiniMax records exist yet in the global log. The Fallback Events panel will display the empty state on first render — this is correct and expected behavior. MiniMax data will populate when Phase 9–13 features are exercised.

## Open Questions

1. **Should `fallback_reason` be a new field in the [CODEX_RESULT] marker?**
   - What we know: `codex-handoff.js` currently emits `source: 'api-fallback'` but no explicit reason string. The reason is the Codex CLI failure (rate limit), inferred from `rate_limit_pct >= 95` or the error string in `codexResult`.
   - What's unclear: Whether the planner should add a `fallback_reason` field to `codex-handoff.js` as a small Phase 13 extension, or infer it from `rate_limit_pct` in Phase 14 dashboard rendering.
   - Recommendation: Infer from existing fields. `rate_limit_pct >= 95` → "Rate limit". Otherwise → "Codex failure". Avoids touching Phase 13 code and keeps Phase 14 self-contained.

2. **Pie chart for model cost breakdown (D-01)**
   - What we know: Chart.js 4.5.1 UMD includes the `pie` and `doughnut` chart types natively. No extra download needed.
   - What's unclear: CONTEXT.md says "Cost breakdown pie chart shows Opus vs Codex vs MiniMax split." But the existing `cost` total does not include Opus (Opus costs are not in the token log). The "Opus" slice would have to be `opusBaseline - actualCost` (the savings amount) to represent the counterfactual, which is conceptually misleading.
   - Recommendation: Implement as a model-cost breakdown (Codex cost vs MiniMax cost), with Opus shown as "baseline" in a separate summary card. Label it "Actual Cost by Model" not "Opus vs Codex vs MiniMax."

## Validation Architecture

Nyquist validation is explicitly disabled in `.planning/config.json` (`"nyquist_validation": false`). This section is omitted per the skip condition.

## Sources

### Primary (HIGH confidence)
- Direct code inspection of `/home/alucard/.claude/hooks/codex-token-logger.js` — Current logger field list, [CODEX_RESULT] parsing
- Direct code inspection of `/home/alucard/.claude/hooks/codex-cost-reporter.js` — `generateReport()` structure, byModel grouping
- Direct code inspection of `/home/alucard/.claude/hooks/codex-global-aggregator.js` — processFile(), enrichment, D-08 confirmation (no changes needed)
- Direct code inspection of `/home/alucard/.claude/hooks/codex-dashboard-generator.js` — `computeMetrics()` modelSplit, Chart.js buildLine/buildBar patterns, blockLog pattern
- Direct code inspection of `/home/alucard/.claude/hooks/codex-pricing.js` — MiniMax pricing already present ($0.30/$0.06/$1.20 per 1M)
- Direct code inspection of `/home/alucard/.claude/hooks/codex-handoff.js` — [CODEX_RESULT] marker fields emitted for 'api-fallback' source
- `~/.claude/dashboard/global.jsonl` — 222 records, all `gpt-5.4`, no MiniMax data yet (confirms empty state will trigger on first render)
- `.planning/phases/14-three-model-reporting/14-CONTEXT.md` — All locked decisions D-01 through D-10

### Secondary (MEDIUM confidence)
- `.planning/STATE.md` — Accumulated decisions from Phases 5-13 confirming established patterns
- `.planning/phases/13-codex-execution-pipeline/13-CONTEXT.md` — D-12 confirms fallback logging via `source: 'cli-fallback'`/`'api-fallback'`

## Metadata

**Confidence breakdown:**
- Token logger changes: HIGH — code inspected, field mapping is clear
- Cost reporter changes: HIGH — `byModel` loop already handles dynamic models, extension is mechanical
- Global aggregator changes: HIGH — confirmed NO changes needed (D-08)
- Dashboard generator changes: HIGH — Chart.js patterns verified in existing code; modelSplit and blockLog patterns directly reusable
- Fallback reason inference: MEDIUM — pragmatic choice (infer from existing fields vs. new field); both approaches valid

**Research date:** 2026-04-03
**Valid until:** 2026-05-03 (stable Node.js/Chart.js stack; 30-day validity)
