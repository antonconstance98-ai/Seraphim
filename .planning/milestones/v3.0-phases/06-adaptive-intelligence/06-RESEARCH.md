# Phase 6: Adaptive Intelligence - Research

**Researched:** 2026-04-08
**Domain:** Statistical pattern analysis, dashboard panel generation, human-approval recommendation workflow
**Confidence:** HIGH

---

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions
- **D-01:** Always analyze from the very first pipeline run. Never wait for a threshold.
- **D-02:** Recommendations labeled with statistical confidence: LOW / MEDIUM / HIGH based on sample size and signal strength (e.g., p-value or effect size).
- **D-03:** Never auto-apply ANY recommendation regardless of confidence level. All changes surface to the user for explicit approval. Low-confidence findings are informational only.
- **D-04:** Rejected recommendations are logged with timestamp and reason for audit trail.
- **D-05:** New Seraphim-branded dashboard — separate from existing `~/.claude/dashboard/dashboard.html`. Own path, own identity, can evolve independently.
- **D-06:** Dashboard location TBD by Claude (could be `~/.seraphim/dashboard/` or `~/.claude/plugins/seraphim/dashboard/`).
- **D-07:** Analysis runs automatically after every complete pipeline run (triggered after Crucible phase completes). No manual invocation needed for standard analysis.
- **D-08:** A `/seraphim:analyze` command also available for on-demand deep analysis outside the pipeline flow.

### Claude's Discretion
- Statistical methods for confidence scoring (simple rates vs z-tests vs bayesian)
- Dashboard technology (Chart.js inline like existing, or evolve if Phase 7 Vercel hosting changes the approach)
- Recommendation presentation format in terminal

### Deferred Ideas (OUT OF SCOPE)
- None — discussion stayed within phase scope
</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| ADPT-01 | Pattern analysis engine reads decisions.jsonl and computes per-phase model rejection rates, cost/quality ratios, and rolling performance averages | decisions.jsonl schema fully understood; fields provide all required signals; pure Node.js implementation, no dependencies |
| ADPT-02 | Auto-recommendation system suggests profile or override changes based on accumulated data | Recommendation logic is deterministic rule application over aggregated metrics; example output exists in design spec |
| ADPT-03 | All recommendations require explicit human approval — never auto-applied; rejected recommendations logged for audit | Approved pattern: print to terminal at session end / analyze run; user applies via `/seraphim:override` or config edit; rejection stored in decisions.jsonl with `type: "recommendation_response"` |
| ADPT-04 | Per-phase model performance heatmap panel added to dashboard showing success rates by (model, phase) combination | Chart.js 4.5.1 already cached; existing generator pattern (computeMetrics + buildHTML) directly reusable; heatmap via matrix dataset |
| ADPT-05 | Profile cost/quality comparison panel in dashboard showing average cost per run vs Crucible pass rate per profile | Cost and crucible_pass_rate both present in decisions.jsonl; aggregation grouping by profile is straightforward |
| ADPT-06 | Recommendation log panel showing suggested changes, user responses (accepted/rejected/ignored), and outcome after acceptance | Recommendation records stored in decisions.jsonl with `type` field; log panel is table rendering of those records |
</phase_requirements>

---

## Summary

Phase 6 delivers three tightly coupled components: (1) a pattern analysis engine that reads `.seraphim/decisions.jsonl` and produces structured recommendations, (2) a human-approval workflow for those recommendations, and (3) a Seraphim-branded HTML dashboard with three panels. All three build directly on infrastructure that Phase 4 created — the decisions.jsonl schema is fully defined and validated, and the existing codex dashboard generator provides a proven pattern for self-contained Chart.js HTML generation.

The statistical methods required are simple and defensible. This is a single-user system with potentially 20-100+ records — not big-data scale. Simple rate calculations (rejection_rate = rejected_runs / total_runs), rolling averages over last N records, and confidence labels based on sample size thresholds (LOW < 5 samples, MEDIUM 5–19, HIGH >= 20) are both sufficient and honest. No ML, no z-tests, no Bayesian inference — those are over-engineering for this data volume and violate the FUTR-05 anti-feature explicitly called out in REQUIREMENTS.md.

The dashboard should live at `~/.claude/plugins/seraphim/dashboard/seraphim.html`. This keeps all plugin assets co-located, makes it easy to find, and keeps a clean boundary from the existing `~/.claude/dashboard/dashboard.html`. Chart.js 4.5.1 is already cached at `~/.claude/dashboard/assets/chart.min.js` and can be read from that location or copied on first generation.

**Primary recommendation:** Build `lib/pattern-analyzer.js` (pure functions, no external deps), `lib/recommendation-engine.js` (rules over aggregated metrics), `lib/dashboard-generator.js` (seraphim-flavored fork of the existing codex generator), and two command files (`analyze.md`, `recommendations.md`). Hook pattern-analyzer + recommendation surface into the existing `crucible.md` command's final step to satisfy D-07.

---

## Standard Stack

### Core
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| Node.js built-ins (`fs`, `path`, `crypto`, `http`) | v22.22.0 (installed) | File I/O, dashboard HTML generation | Zero new dependencies; matches all existing plugin code; already used in every lib file |
| Chart.js | 4.5.1 UMD (cached at `~/.claude/dashboard/assets/chart.min.js`) | Dashboard charting | Already present and integrity-verified in existing dashboard; no re-download needed |

### Supporting
| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| None required | — | — | All analysis is vanilla JS operating over JSONL lines |

### Alternatives Considered
| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| Vanilla JS statistics | simple-statistics npm | simple-statistics adds a 40KB dependency for functions that are 10-line implementations at this data scale; not worth it |
| Chart.js heatmap plugin (chartjs-chart-matrix) | Custom table-based heatmap with CSS | The matrix plugin adds a dependency; a CSS grid table is simpler and works without CDN; however chartjs-chart-matrix is the correct tool if Chart.js is already present — evaluate at implementation |

**Installation:** No new packages needed. All logic uses Node.js stdlib.

---

## Architecture Patterns

### Recommended Project Structure

New files created in Phase 6:

```
~/.claude/plugins/seraphim/
├── lib/
│   ├── pattern-analyzer.js      # aggregateDecisions(), computeRejectionRates(), computeCostQuality()
│   ├── recommendation-engine.js # generateRecommendations(metrics) -> [{type, phase, model, reason, confidence}]
│   └── dashboard-generator.js   # generateSeraphimDashboard(data) -> HTML string; atomic write
├── commands/
│   ├── analyze.md               # /seraphim:analyze — on-demand deep analysis (D-08)
│   └── recommendations.md       # /seraphim:recommendations — display pending + history
└── dashboard/
    └── seraphim.html            # generated output; never committed (runtime artifact)
```

Changes to existing files:

```
~/.claude/plugins/seraphim/commands/crucible.md
  └── Step N+1: after crucible completes, call pattern-analyzer + surface recommendations (D-07)
```

### Pattern 1: JSONL Aggregation — Grouping decisions by (phase, model)

**What:** Read all records from decisions.jsonl, group by (phase, model), compute per-group metrics.
**When to use:** Core of ADPT-01 — every dashboard panel and recommendation derives from this aggregation.

```javascript
// Source: Pattern derived from ~/.claude/hooks/codex-global-aggregator.js (verified on disk)
'use strict';
const fs = require('fs');

function loadDecisions(projectRoot) {
  const p = require('path').join(projectRoot, '.seraphim', 'decisions.jsonl');
  if (!fs.existsSync(p)) return [];
  return fs.readFileSync(p, 'utf8')
    .split('\n')
    .filter(l => l.trim())
    .map(l => { try { return JSON.parse(l); } catch { return null; } })
    .filter(Boolean);
}

function groupBy(records, keyFn) {
  return records.reduce((acc, r) => {
    const k = keyFn(r);
    if (!acc[k]) acc[k] = [];
    acc[k].push(r);
    return acc;
  }, {});
}

function computeRejectionRates(records) {
  // Rejection = outcome is 'failure' or loop_count > 0 on judge/crucible phases
  const phaseModelGroups = groupBy(
    records.filter(r => r.type !== 'recommendation' && r.type !== 'recommendation_response'),
    r => `${r.phase}::${r.model}`
  );
  return Object.entries(phaseModelGroups).map(([key, recs]) => {
    const [phase, model] = key.split('::');
    const n = recs.length;
    const failures = recs.filter(r => r.outcome === 'failure' || r.loop_count > 0).length;
    return {
      phase, model, n,
      rejection_rate: n > 0 ? failures / n : 0,
      confidence: n < 5 ? 'LOW' : n < 20 ? 'MEDIUM' : 'HIGH'
    };
  });
}
```

### Pattern 2: Recommendation Generation — Rule-based thresholds

**What:** Apply threshold rules over aggregated metrics to generate human-readable recommendation objects.
**When to use:** ADPT-02. Recommendations are purely deterministic — no ML, no probabilistic sampling.

```javascript
// Threshold rules (values are starting proposals; adjust after first real data)
const RULES = [
  {
    id: 'high-rejection-rate',
    test: (m) => m.rejection_rate >= 0.6 && m.n >= 3,
    message: (m) => `${m.model} in ${m.phase} rejected ${Math.round(m.rejection_rate * 100)}% of runs — consider switching to a higher-quality model for this phase`,
    confidence: (m) => m.confidence
  },
  {
    id: 'low-crucible-pass-rate',
    // crucible_pass_rate is in quality_signals; compute average across phase records
    test: (m) => m.avg_crucible_pass_rate !== null && m.avg_crucible_pass_rate < 0.5 && m.n >= 3,
    message: (m) => `${m.model} in ${m.phase} Crucible pass rate is ${Math.round(m.avg_crucible_pass_rate * 100)}% — consider upgrading`,
    confidence: (m) => m.confidence
  }
];

function generateRecommendations(metrics) {
  const recs = [];
  for (const m of metrics) {
    for (const rule of RULES) {
      if (rule.test(m)) {
        recs.push({
          type: 'recommendation',
          timestamp: new Date().toISOString(),
          rule_id: rule.id,
          phase: m.phase,
          model: m.model,
          message: rule.message(m),
          confidence: rule.confidence(m),
          status: 'pending',  // pending | accepted | rejected | ignored
          response_timestamp: null,
          response_reason: null
        });
      }
    }
  }
  return recs;
}
```

### Pattern 3: Recommendation Storage — Append to decisions.jsonl

**What:** Recommendations and their responses are stored as typed records in the same decisions.jsonl file, using the `type` field.
**When to use:** ADPT-03, ADPT-06. Keeps one data file; `decisions-validator.js` already handles multi-type records gracefully (only validates REQUIRED_FIELDS which have no `type` check).

```javascript
// Recommendation record — appended after analysis
{
  "type": "recommendation",
  "timestamp": "2026-04-08T...",
  "rule_id": "high-rejection-rate",
  "phase": "envision",
  "model": "qwen-3.5-27b",
  "message": "Qwen Envision rejected 4/5 runs...",
  "confidence": "HIGH",
  "status": "pending"
}

// Response record — appended when user acts (accepted/rejected/ignored)
{
  "type": "recommendation_response",
  "timestamp": "2026-04-08T...",
  "rule_id": "high-rejection-rate",
  "phase": "envision",
  "original_recommendation_timestamp": "2026-04-08T...",
  "response": "rejected",          // accepted | rejected | ignored
  "reason": "Qwen is free, will retry"
}
```

**Critical:** `decisions-validator.js` checks REQUIRED_FIELDS (`timestamp`, `phase`, `model`, ...). Recommendation records are missing `model`, `profile`, etc. The validator must be extended to skip records where `record.type` is `"recommendation"` or `"recommendation_response"`. This is a Phase 6 task.

### Pattern 4: Dashboard Generation — Self-contained HTML with inlined Chart.js

**What:** Generate a single `.html` file with Chart.js inlined, three panel sections, atomic write.
**When to use:** ADPT-04, ADPT-05, ADPT-06. Follow existing `codex-dashboard-generator.js` pattern exactly.

```javascript
// Source: Pattern from ~/.claude/hooks/codex-dashboard-generator.js (verified on disk)
// Atomic write prevents partial-write corruption
const tmp = HTML_PATH + '.tmp.' + process.pid;
fs.writeFileSync(tmp, html, 'utf8');
fs.renameSync(tmp, HTML_PATH);  // atomic on same filesystem
```

### Pattern 5: Heatmap Panel — CSS Grid table (no extra Chart.js plugin)

**What:** Per-phase model performance heatmap. Use an HTML/CSS table with background-color interpolated from 0% (red) to 100% (green) rejection rate. Avoids chartjs-chart-matrix dependency.
**When to use:** ADPT-04. Simple and reliable.

```javascript
// Color from rate (0=green, 1=red)
function rateToColor(rate) {
  const r = Math.round(rate * 220);
  const g = Math.round((1 - rate) * 180);
  return `rgb(${r},${g},60)`;
}
```

### Anti-Patterns to Avoid

- **Auto-applying recommendations in code:** Any code path that modifies `.seraphim/config.json` based on a recommendation without explicit user invocation violates D-03. The analysis engine must only append records and print to terminal.
- **Using ML or scipy-style statistics:** FUTR-05 is explicit — ML at single-user scale is insufficient data. Use rates and rolling averages only.
- **Extending `~/.claude/dashboard/dashboard.html`:** D-05 locks the Seraphim dashboard to a new file. Do not add panels to the existing Codex dashboard.
- **Storing recommendations in a separate file:** Keeping a second JSONL file adds schema drift risk. Extend decisions.jsonl with a `type` field discriminator.
- **Checking `type` field in decisions-validator.js for REQUIRED_FIELDS:** The validator must skip type=recommendation records, not fail on them.

---

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Chart.js integration | Custom canvas drawing | Chart.js 4.5.1 (already cached) | Cached, integrity-verified, used in existing dashboard |
| Atomic file writes | Manual write + rename | `fs.writeFileSync(tmp); fs.renameSync(tmp, final)` pattern | Already proven in codex-dashboard-generator.js; prevents partial-write corruption |
| JSONL parsing | Custom tokenizer | `fs.readFileSync().split('\n').map(JSON.parse)` | Sufficient; decisions.jsonl is line-delimited, no multi-line values |
| Run grouping | Timestamp heuristics | Phase=`discover` record boundary (already established in history.md) | history.md already solves this; copy the grouping logic |

**Key insight:** The entire pattern analysis engine can be built with zero npm dependencies. The data is JSONL, the statistics are rates and averages, the output is HTML. Node.js stdlib handles everything.

---

## Common Pitfalls

### Pitfall 1: decisions-validator.js rejects recommendation records
**What goes wrong:** Session start runs `validateDecisions()`; recommendation records are missing `model`, `profile`, `tokens_in`, etc.; validator throws violations; user sees spurious integrity errors every session.
**Why it happens:** REQUIRED_FIELDS was defined before `type`-discriminated records existed.
**How to avoid:** Add a guard at the top of the per-record validation loop: `if (record.type === 'recommendation' || record.type === 'recommendation_response') return;`
**Warning signs:** Integrity violations showing `missing required field` for records with `type: "recommendation"`.

### Pitfall 2: Heatmap cells empty when no data exists for a (model, phase) pair
**What goes wrong:** Dashboard shows blank or broken heatmap when certain model+phase combinations have no records yet.
**Why it happens:** Aggregation returns undefined for missing pairs.
**How to avoid:** Use a complete matrix initializer — iterate all known phases × all known models from `models.json`; default empty cells to `{ n: 0, rejection_rate: null }` rendered as gray.

### Pitfall 3: Duplicate recommendations generated every analysis run
**What goes wrong:** Each time the analyzer runs, it regenerates recommendations without checking if an equivalent pending recommendation already exists.
**Why it happens:** The engine generates recommendations from metrics without deduplication.
**How to avoid:** Before appending a new recommendation, check if an existing record with the same `rule_id + phase + model` and `status: "pending"` already exists. Only append if none exists.

### Pitfall 4: Dashboard generation fails silently when Chart.js cache is missing
**What goes wrong:** `~/.claude/dashboard/assets/chart.min.js` may not exist on a fresh install; dashboard HTML renders blank.
**Why it happens:** The existing dashboard generator creates this file, but if Phase 5 was never run, the cache doesn't exist.
**How to avoid:** Include an `ensureChartJs()` step (copy the verified pattern from codex-dashboard-generator.js) or bundle Chart.js inline via CDN `<script src="...">` fallback. Prefer inline for offline safety.

### Pitfall 5: Analyze command triggers for incomplete runs
**What goes wrong:** Analysis fires after Crucible even when the run only completed 2 of 6 phases (e.g., user ran just `/seraphim:judge`).
**Why it happens:** D-07 says "after complete pipeline run" but the trigger point in crucible.md fires any time Crucible completes.
**How to avoid:** In the Crucible command's analysis trigger, only fire if records for all six phases exist in the current run grouping. Check via the run-grouping logic: count distinct phase values in the current run.

### Pitfall 6: Recommendation confidence label mismatch with sample sizes
**What goes wrong:** Recommendation says "HIGH confidence" based on 5 samples; user trusts it too much; recommendation is wrong.
**Why it happens:** Threshold for HIGH set too low.
**How to avoid:** Locked thresholds: LOW = n < 5, MEDIUM = 5 ≤ n < 20, HIGH = n ≥ 20. Document these in the engine's constants. LOW-confidence recommendations are labeled "Informational only" per D-02/D-03.

---

## Code Examples

### Run Grouping (already established by history.md)

```javascript
// Source: ~/.claude/plugins/seraphim/commands/history.md (verified on disk)
// Group records into runs: new 'discover' record after any non-discover records starts a new run
const runs = [];
let current = [];
for (const rec of records) {
  if (rec.phase === 'discover' && current.length > 0) {
    runs.push(current);
    current = [];
  }
  current.push(rec);
}
if (current.length > 0) runs.push(current);
```

### Profile Cost/Quality Aggregation (ADPT-05)

```javascript
// Group by profile; compute avg cost per run and avg crucible pass rate
function computeProfileMetrics(runs) {
  const byProfile = {};
  for (const run of runs) {
    const profile = run[0]?.profile || 'unknown';
    if (!byProfile[profile]) byProfile[profile] = { runs: 0, totalCost: 0, cruciblePasses: 0, crucibleRuns: 0 };
    const g = byProfile[profile];
    g.runs++;
    g.totalCost += run.reduce((s, r) => s + (r.cost_usd || 0), 0);
    const crucibleRec = run.find(r => r.phase === 'crucible');
    if (crucibleRec && crucibleRec.quality_signals?.crucible_pass_rate !== null) {
      g.crucibleRuns++;
      g.cruciblePasses += crucibleRec.quality_signals.crucible_pass_rate || 0;
    }
  }
  return Object.entries(byProfile).map(([profile, g]) => ({
    profile,
    avg_cost: g.runs > 0 ? g.totalCost / g.runs : 0,
    avg_crucible_pass_rate: g.crucibleRuns > 0 ? g.cruciblePasses / g.crucibleRuns : null
  }));
}
```

### Terminal Recommendation Display

```
┌─────────────────────────────────────────────────────────────┐
│  Seraphim Analysis — 1 recommendation                       │
├─────────────────────────────────────────────────────────────┤
│  [HIGH] envision / qwen-3.5-27b                             │
│  Qwen Envision rejected 4/5 runs — consider Gemini 3.1 Pro  │
│                                                             │
│  To apply:   /seraphim:override envision gemini-3.1-pro     │
│  To dismiss: /seraphim:recommendations dismiss <id>         │
└─────────────────────────────────────────────────────────────┘
```

### Dashboard Location Decision

Use `~/.claude/plugins/seraphim/dashboard/seraphim.html`. Rationale:
- Co-located with plugin source — easy to find, version together
- Clean separation from `~/.claude/dashboard/dashboard.html` (satisfies D-05)
- Phase 7 (multi-project dashboard) can extend or replace this file without touching the Codex dashboard

---

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Single global dashboard extended for everything | Separate branded dashboards per tool | Phase 6 (D-05) | Seraphim dashboard is independent; Phase 7 can evolve it for multi-project without breaking Codex dashboard |
| Wait for N samples before analyzing | Analyze from run 1, label LOW confidence | D-01 | No cold-start delay; user gets informational signal immediately |

---

## Open Questions

1. **Does `decisions-validator.js` need backward-compatible schema evolution?**
   - What we know: Currently validates REQUIRED_FIELDS for every record with no type check
   - What's unclear: Whether existing records in decisions.jsonl (if any) need migration
   - Recommendation: Add type-guard skip at top of per-record validation; no migration needed since recommendation records are new in Phase 6

2. **chartjs-chart-matrix plugin availability offline**
   - What we know: Plugin exists (`npm install chartjs-chart-matrix`); adds ~15KB
   - What's unclear: Whether the matrix plugin is needed or a CSS grid table is better
   - Recommendation: Use CSS grid heatmap table — zero dependency, works offline, fully controllable styling; simpler to generate from template literal

3. **`/seraphim:recommendations dismiss <id>` — how are recommendation IDs assigned?**
   - What we know: Recommendations are appended to decisions.jsonl with a timestamp
   - What's unclear: Whether a short ID (e.g., sequential integer) is needed for terminal UX
   - Recommendation: Use a short hash of `rule_id + phase + model + timestamp` (first 8 chars) as the display ID; stored in the recommendation record as `rec_id`

---

## Environment Availability

| Dependency | Required By | Available | Version | Fallback |
|------------|------------|-----------|---------|----------|
| Node.js | All lib files | Yes | v22.22.0 | — |
| Chart.js 4.5.1 (cached) | Dashboard HTML | Yes (at `~/.claude/dashboard/assets/chart.min.js`) | 4.5.1 | Download from CDN if missing |
| decisions.jsonl | Pattern analysis | Exists only after Phase 4 runs have been executed | N/A | Analyzer exits gracefully with "No data yet" message |

---

## Validation Architecture

> nyquist_validation not explicitly disabled; treating as enabled.

### Test Framework
| Property | Value |
|----------|-------|
| Framework | Node.js built-in `assert` (no test framework installed; consistent with existing plugin) |
| Config file | none |
| Quick run command | `node -e "require('./lib/pattern-analyzer'); console.log('OK')"` |
| Full suite command | `node ~/.claude/plugins/seraphim/tests/phase-06-smoke.js` |

### Phase Requirements → Test Map
| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| ADPT-01 | computeRejectionRates returns correct rates for fixture data | unit | `node tests/test-pattern-analyzer.js` | ❌ Wave 0 |
| ADPT-02 | generateRecommendations returns correct recommendation for high rejection rate fixture | unit | `node tests/test-recommendation-engine.js` | ❌ Wave 0 |
| ADPT-03 | Recommendation records appended; response records appended; validator skips them | unit | `node tests/test-decisions-validator-compat.js` | ❌ Wave 0 |
| ADPT-04 | Dashboard HTML contains heatmap table with correct cell count | unit | `node tests/test-dashboard-generator.js` | ❌ Wave 0 |
| ADPT-05 | Profile metrics computation correct for multi-profile fixture | unit | `node tests/test-dashboard-generator.js` | ❌ Wave 0 |
| ADPT-06 | Recommendation log panel renders all fixture recommendation records | unit | `node tests/test-dashboard-generator.js` | ❌ Wave 0 |

### Wave 0 Gaps
- [ ] `tests/test-pattern-analyzer.js` — covers ADPT-01
- [ ] `tests/test-recommendation-engine.js` — covers ADPT-02
- [ ] `tests/test-decisions-validator-compat.js` — covers ADPT-03 (validator backward compat)
- [ ] `tests/test-dashboard-generator.js` — covers ADPT-04, ADPT-05, ADPT-06
- [ ] `tests/fixtures/decisions-fixture.jsonl` — sample data for all tests

---

## Sources

### Primary (HIGH confidence)
- `~/.claude/plugins/seraphim/lib/decisions-logger.js` — decisions.jsonl schema (verified on disk 2026-04-08)
- `~/.claude/plugins/seraphim/lib/decisions-validator.js` — validation logic (verified on disk 2026-04-08)
- `~/.claude/plugins/seraphim/config/models.json` — model roster (verified on disk 2026-04-08)
- `~/.claude/plugins/seraphim/config/profiles.json` — profile definitions (verified on disk 2026-04-08)
- `~/.claude/hooks/codex-dashboard-generator.js` — HTML generation pattern (verified on disk 2026-04-08)
- `~/.claude/plugins/seraphim/commands/history.md` — run grouping logic (verified on disk 2026-04-08)
- `.planning/phases/06-adaptive-intelligence/06-CONTEXT.md` — locked decisions (verified on disk 2026-04-08)
- `docs/specs/2026-04-04-seraphim-v3-design.md` §Adaptive Intelligence — design spec (verified on disk 2026-04-08)
- `.planning/research/FEATURES.md` §Adaptive Model Selection — feature analysis (verified on disk 2026-04-08)

### Secondary (MEDIUM confidence)
- REQUIREMENTS.md ADPT-01 through ADPT-06 — requirement text (verified on disk 2026-04-08)

---

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH — all libraries already present on disk; no new dependencies
- Architecture: HIGH — patterns verified directly in existing plugin source code
- Pitfalls: HIGH — derived from reading the existing validator, logger, and dashboard generator code directly

**Research date:** 2026-04-08
**Valid until:** 2026-05-08 (stable; no fast-moving dependencies)
