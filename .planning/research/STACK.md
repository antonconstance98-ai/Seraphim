# Technology Stack

**Project:** Claude X Codex — v1.1 Global Metrics Dashboard
**Researched:** 2026-04-02
**Confidence:** HIGH (verified against live system, official docs, and installed versions)

---

## Existing Stack (DO NOT RE-RESEARCH)

All v1.0 components are validated and unchanged:

| Technology | Version | Purpose |
|------------|---------|---------|
| Node.js | v22.22.0 | All hook scripts |
| npm | 10.9.4 | Package management |
| `@openai/codex` CLI | 0.118.0 | Codex execution |
| `openai` npm package | 6.33.0 (at `~/.npm-global/lib/node_modules/`) | API calls |
| Claude Code hooks | v2.1.90 | SessionStart, PostToolUse, Stop, SubagentStop |

---

## New Stack for v1.1 Dashboard

### Core: Zero New npm Packages Required

All new capabilities use Node.js v22 built-in APIs or static file embedding. No `npm install` step is needed for the dashboard.

---

### Component 1: Global JSONL Aggregation

**Approach:** Node.js v22 built-in `fs.glob()` async iterator

| API | Source | Purpose | Why |
|-----|--------|---------|-----|
| `fs/promises` `.glob()` | Node.js v22 built-in | Recursively find all `token-log.jsonl` files across `~/projects/` and `~/.claude/` | Confirmed working on this machine (tested live). Returns async iterator. Zero dependencies. Replaces the need for `glob` or `fast-glob` npm packages entirely. |
| `fs/promises` `.readFile()` | Node.js v22 built-in | Read each discovered JSONL file | Already used in existing hooks |

**Verified API usage pattern (tested live on v22.22.0):**
```javascript
const fs = require('node:fs/promises');
const os = require('os');

// Returns AsyncIterator — must use for-await-of, NOT .then()
const files = [];
for await (const f of fs.glob('**/.planning/token-log.jsonl', {
  cwd: os.homedir()
})) {
  files.push(f);
}
```

**Do NOT use:** `glob` npm package (v13.0.6 latest — unnecessary), `fast-glob` npm package — both are superseded by the Node.js built-in on v22.

**Confidence:** HIGH — tested live, returns correct results.

---

### Component 2: HTML Dashboard Generation

**Approach:** ES6 template literals in a Node.js script — no templating engine

| Mechanism | Why |
|-----------|-----|
| ES6 template literals (backtick strings) | The entire dashboard is a single `generateHTML(data)` function returning a string. No separate template files, no partials, no build step. Existing hooks use this exact pattern for generating Markdown reports (see `codex-cost-reporter.js`). |

**Do NOT use:**
- EJS, Mustache, Eta, Handlebars — these add an npm dependency for no benefit when the template is a single function in one file
- React, Vue, Svelte — require a build step and are completely inappropriate here
- Nunjucks, Pug — same issue as EJS; template engines are for Express servers, not static file generation

**Pattern (matches existing `codex-cost-reporter.js` style):**
```javascript
function generateDashboardHTML(aggregatedData) {
  const { projects, totals, dailyTrend, chartJsSource } = aggregatedData;
  return `<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>Codex Dashboard</title>
  <style>${inlineCSS}</style>
</head>
<body>
  <script>${chartJsSource}</script>
  <script>${dashboardJS}</script>
</body>
</html>`;
}
```

**Confidence:** HIGH — established pattern in this codebase.

---

### Component 3: Charts in the Generated Dashboard

**Approach:** Chart.js v4.5.1 UMD build, downloaded once and cached as a static file, inlined into every generated HTML file

| Technology | Version | File Size | Distribution |
|------------|---------|-----------|-------------|
| Chart.js | 4.5.1 (latest stable) | 204 KB uncompressed | UMD minified from jsDelivr CDN |

**Why Chart.js over alternatives:**

| Option | Size | Bar + Line Support | API Complexity | Decision |
|--------|------|-------------------|----------------|---------|
| **Chart.js 4.5.1** | **204 KB** | **Yes — first class** | **Low — `new Chart(ctx, config)`** | **CHOSEN** |
| uPlot 1.6.32 | 50 KB | Bar chart is undocumented/hacky | High — poor docs | Rejected: Bar charts require plugins with sparse documentation; not worth for 4x smaller size when file is never transferred over network |
| D3.js | ~500 KB | Yes, but requires drawing primitives manually | Very high | Rejected: massive overkill for 3 chart types |
| Pure SVG (no library) | 0 KB | Yes, but requires manual coordinate math | Low code complexity, high math complexity | Viable fallback if Chart.js causes issues; acceptable for simple bar charts but time-series line charts are significantly harder |

**Chart types needed:**
- Line chart: daily cost + savings over time (time-series x-axis)
- Bar chart: per-project cost breakdown (categorical x-axis)
- Stacked bar: token breakdown by model per session (optional)

Chart.js handles all three with identical `new Chart(canvas, { type: 'line'|'bar', ... })` API.

**How Chart.js is integrated (no npm install):**

The dashboard generator script (`codex-dashboard-generator.js`) downloads the Chart.js UMD build once on first run:

```javascript
const CHART_JS_URL = 'https://cdn.jsdelivr.net/npm/chart.js@4.5.1/dist/chart.umd.min.js';
const CHART_JS_CACHE = path.join(os.homedir(), '.claude', 'dashboard', 'assets', 'chart.umd.min.js');

async function getChartJsSource() {
  // Return cached copy if it exists
  if (fs.existsSync(CHART_JS_CACHE)) {
    return fs.readFileSync(CHART_JS_CACHE, 'utf8');
  }
  // Download once and cache
  const source = await fetch(CHART_JS_URL).then(r => r.text());
  fs.mkdirSync(path.dirname(CHART_JS_CACHE), { recursive: true });
  fs.writeFileSync(CHART_JS_CACHE, source, 'utf8');
  return source;
}
```

The returned source string is then embedded as `<script>` content in the HTML — no CDN call at dashboard open time.

**Note:** `fetch()` is Node.js v18+ built-in. On v22.22.0 it is available without import.

**Confidence:** HIGH — Chart.js 4.5.1 confirmed as latest via `npm show chart.js version`. File size verified by downloading the actual file (204,390 bytes). uPlot version 1.6.32 confirmed via npm.

---

### Component 4: Dashboard Output Location

| Path | Purpose |
|------|---------|
| `~/.claude/dashboard/index.html` | The generated dashboard — opened in browser |
| `~/.claude/dashboard/assets/chart.umd.min.js` | Cached Chart.js source — embedded inline at generation time |

The SessionStart hook triggers `codex-dashboard-generator.js`, which:
1. Scans all projects for `token-log.jsonl` files
2. Aggregates all records into cross-project stats
3. Generates `~/.claude/dashboard/index.html` with inlined Chart.js

---

### Component 5: Session Tracking Integration

**Approach:** Extend existing JSONL schema — no new fields required for MVP

The existing `token-log.jsonl` record already contains:
```json
{
  "timestamp": "2026-04-02T22:52:30.644Z",
  "session_id": "30092b41-eaa3-45bf-95b4-64463d5a2dbd",
  "model": "gpt-5.4",
  "source": "cli",
  "task_type": "review",
  "tokens": { "input": 12860, "cached_input": 10624, "output": 365 },
  "cost_usd": 0.04908
}
```

The `session_id` field enables session history grouping. The `timestamp` field enables daily/weekly time bucketing. **No schema changes needed.**

The aggregator groups records by `session_id` for session history and by date truncation (`timestamp.slice(0, 10)`) for time trends.

**Confidence:** HIGH — schema verified from live `token-log.jsonl` file on this machine.

---

## Installation

No `npm install` step is needed for new dashboard capabilities. All dependencies are:
- Node.js v22 built-ins (`fs/promises`, `path`, `os`, `fetch`)
- Chart.js downloaded once on first dashboard generation

```bash
# Verify prerequisites (all already satisfied)
node --version    # Must be v18+ for built-in fetch; v22.22.0 confirmed

# No npm install required.

# First dashboard generation downloads Chart.js automatically.
# Dashboard location after first run:
# ~/.claude/dashboard/index.html
```

---

## Alternatives Considered

| Category | Recommended | Alternative | Why Not |
|----------|-------------|-------------|---------|
| File discovery | Node.js v22 built-in `fs.glob()` | `glob` npm package (v13.0.6) | Node.js v22 has this built-in; adding a package for a built-in is unnecessary |
| File discovery | Node.js v22 built-in `fs.glob()` | `fast-glob` npm package | Same reason; `fast-glob` is faster for 200k+ files, irrelevant at <100 project directories |
| HTML generation | ES6 template literals | EJS / Mustache / Eta | These are for Express apps rendering separate template files; a single `generateHTML()` function needs no template engine |
| Chart library | Chart.js 4.5.1 (inlined) | uPlot 1.6.32 (inlined) | uPlot is 4x smaller but its bar chart support requires undocumented plugins; the 150KB size difference is irrelevant for a local file |
| Chart library | Chart.js 4.5.1 (inlined) | Pure SVG via template literals | Viable for bar charts but time-series line charts require manual coordinate math and axis scaling — not worth the complexity |
| Chart integration | Download once, embed inline | CDN `<script>` tag | Dashboard must work offline; CDN link fails if not connected |
| Chart integration | Download once, embed inline | `chartjs-node-canvas` npm package | That package renders charts as PNG images server-side; we want interactive JS charts in the browser |

---

## What NOT to Use

| Technology | Reason | Use Instead |
|------------|--------|-------------|
| `chartjs-node-canvas` | Renders charts as PNG in Node.js — requires `canvas` C binding, creates static images, no interactivity | Chart.js UMD build inlined in HTML for interactive browser charts |
| EJS / Mustache / Eta | Template engines for multi-file Express apps; adds a dependency for zero benefit when HTML generation is a single function | ES6 template literals |
| `glob` / `fast-glob` npm packages | Replaced by Node.js v22 built-in `fs.glob()` | `fs/promises` built-in |
| D3.js | 500KB, requires drawing primitives manually; the complexity is for custom data journalism visualisations, not 3-chart dashboards | Chart.js 4.5.1 |
| Any web framework (Express, Fastify, Koa) | Dashboard is a static local file; no server needed, no server allowed | Static HTML file generation |
| `http-server` / `serve` npm packages | Same reason — no server needed | Open the HTML file directly in browser |
| React / Vue / Svelte | Require build step; framework-sized dependency for a dashboard that has no user interaction beyond chart tooltips | Vanilla JS with Chart.js |

---

## Version Compatibility

| Package | Version | Node.js Requirement | Status |
|---------|---------|--------------------|----|
| `fs/promises` glob | Node.js v22 built-in | v22.0.0+ (async iterator form) | CONFIRMED working on v22.22.0 |
| `fetch()` built-in | Node.js v18+ built-in | v18.0.0+ | CONFIRMED on v22.22.0 |
| Chart.js | 4.5.1 | Browser only (the inlined script) | CONFIRMED latest stable |
| uPlot | 1.6.32 (not used) | Browser only | Listed for reference |

**Note on `fs.glob()` API shape:** On Node.js v22, `fs.glob()` returns an **AsyncIterator** (not a Promise). Use `for await...of`, not `.then()`. This was verified by live testing — the `.then()` pattern fails with `TypeError: glob(...).then is not a function`.

---

## Sources

- Node.js v22 `fs/promises` glob: tested live on v22.22.0 (`node -e "const fs = require('node:fs/promises'); console.log(typeof fs.glob)"` → `function`)
- Chart.js 4.5.1: `npm show chart.js version` → `4.5.1` (latest stable, no `next` pre-release)
- Chart.js UMD file size: downloaded `https://cdn.jsdelivr.net/npm/chart.js@4.5.1/dist/chart.umd.min.js` → 204,390 bytes
- uPlot 1.6.32: `npm show uplot version` → `1.6.32` (latest stable); file size 51,081 bytes confirmed
- Existing JSONL schema: verified from live `~/.../Claude_X_Codex/.planning/token-log.jsonl` (10 records reviewed)
- `fetch()` in Node.js v22: built-in, no import required (part of global scope since v18)
- Chart.js homepage: https://www.chartjs.org/docs/latest/getting-started/installation.html
- uPlot GitHub: https://github.com/leeoniya/uPlot

---
*Stack research for: v1.1 Global Metrics Dashboard (Claude X Codex)*
*Researched: 2026-04-02*

---
---

# Stack Additions: ML-Driven Self-Optimization Milestone

**Researched:** 2026-04-03
**Confidence:** MEDIUM-HIGH (npm versions live-verified; ML library maturity caveats flagged)

This section covers **new additions only** for adaptive intelligence (ML-based auto-tuning). All existing stack components above remain unchanged.

---

## Recommended Stack — New Libraries Only

### Core Technologies

| Technology | Version | Purpose | Why Recommended |
|------------|---------|---------|-----------------|
| `better-sqlite3` | 12.8.0 | Structured decision log storage; time-series queries via SQL | Synchronous API matches hook script execution model — no async overhead in hot paths. Fastest SQLite binding for Node.js. Supports JSON columns, window functions, and date-range queries needed for trend detection. Production-ready — `node:sqlite` is still experimental behind a flag on Node.js 22 and explicitly not recommended for production. |
| `simple-statistics` | 7.8.9 | Statistical analysis: means, stddev, Z-score, linear regression, percentiles | Zero dependencies. Pure JS — no native binaries, no compilation step. Covers 90% of the analysis needed for anomaly detection and trend detection on small datasets (hundreds to low thousands of rows). Actively maintained. |
| `ml-regression` | 6.3.0 | Curve fitting and predictive trend lines (simple linear, polynomial) | Part of the `mljs` ecosystem — the only serious ML suite that is pure JS, no Python, no native bindings, actively maintained. Use for forecasting token cost trends and routing performance over time. |

### Supporting Libraries

| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| `regression` | 2.0.1 | Alternative curve fitting (exponential, logarithmic, power fits) | Use instead of `ml-regression` only if you need exponential or logarithmic curve fitting — `ml-regression` covers linear and polynomial out of the box, but stops there. Minimal API, zero dependencies. |
| `ml-matrix` | pulled transitively | Matrix operations underlying regression | Only reference directly if you implement multi-feature scoring models. Comes in automatically with `ml-regression`. |

### Development Tools

| Tool | Purpose | Notes |
|------|---------|-------|
| `better-sqlite3` REPL (via `node -e`) | Inspect decision log database during development | Query live data: `node -e "const db = require('better-sqlite3')(process.env.HOME+'/.claude/optimization.db'); console.log(db.prepare('SELECT * FROM decisions LIMIT 5').all())"` |
| Existing `ccusage` | Cross-reference JSONL cost data with SQLite decision logs | No new install. Aggregate JSONL first, write summaries to SQLite for ML queries. |

---

## Installation

```bash
# Run from the hooks directory or project root
npm install better-sqlite3@12.8.0
npm install simple-statistics@7.8.9
npm install ml-regression@6.3.0

# regression is optional — only needed for exponential/log curve fits
# npm install regression@2.0.1
```

All three are runtime dependencies (used inside hook scripts), not devDependencies.

---

## Integration Points with Existing Hook Scripts

| Existing Component | New Library | Integration Pattern |
|-------------------|-------------|---------------------|
| JSONL token logs | `better-sqlite3` | Post-session aggregator reads JSONL, writes normalized rows into `~/.claude/optimization.db`. Hook scripts query SQLite for analysis — never raw JSONL scan. |
| `PostToolUse` hooks | `simple-statistics` | After each tool execution, hook reads last N rows from SQLite and computes rolling Z-score. If anomaly detected, injects advisory context via `additionalContext`. |
| Routing decision hooks | `ml-regression` | Weekly (or session-end) batch job fits linear regression on routing outcome metrics. Writes updated routing thresholds back to a `config` table in SQLite. Hook scripts read thresholds at startup. |
| Chart.js HTML dashboard | `better-sqlite3` | Dashboard generator queries SQLite directly instead of scanning JSONL files. Faster and enables date-range filtering via SQL `WHERE timestamp > ?`. |

---

## Alternatives Considered

| Recommended | Alternative | When to Use Alternative |
|-------------|-------------|-------------------------|
| `better-sqlite3` | LevelDB / `classic-level` | Only if write throughput exceeds ~10,000 inserts/second. LevelDB has no SQL — every range query and aggregation requires manual key iteration. At this project's volume (dozens of hook events per session), SQLite's query power is the right trade-off. |
| `better-sqlite3` | Extended append-only JSONL | Acceptable for Phase 1 logging only. As soon as you need range queries ("last 7 days"), percentile aggregations, or correlating routing decisions with outcomes, JSONL becomes a maintenance burden. Migrate to SQLite early. |
| `simple-statistics` | TensorFlow.js | Only if you need trained neural networks with backpropagation. TFjs requires `@tensorflow/tfjs-node`, which pulls in native binaries and adds ~150 MB to node_modules. Complete overkill for Z-score and linear regression on hundreds of rows. |
| `simple-statistics` | `brain.js` | brain.js 2.0.0-beta.24 has been in beta for over a year (last npm publish ~April 2025, confirmed via npm registry). GPU acceleration is irrelevant for CLI hook scripts. Avoid until a stable 2.x releases. |
| `ml-regression` | Python `scikit-learn` via `child_process` | Only if model complexity demands ensemble methods or gradient boosting. Adds a Python runtime dependency, complicates deployment, and introduces 2-5 second subprocess startup in synchronous hook paths. `simple-statistics` + `ml-regression` are sufficient for this dataset size. |
| `better-sqlite3` (sync) | `sqlite` async wrapper | Only if hook scripts are refactored to async-first patterns. Existing infrastructure uses synchronous execution and `additionalContext` injection — synchronous SQLite is the correct match. |

---

## What NOT to Use

| Avoid | Why | Use Instead |
|-------|-----|-------------|
| TensorFlow.js (`@tensorflow/tfjs-node`) | ~150 MB native binary, requires recompilation per Node.js version, complete overkill for statistical trend detection on <10,000 rows | `simple-statistics` + `ml-regression` |
| `brain.js` | Stuck at beta (2.0.0-beta.24, last release ~April 2025, confirmed from npm registry). API changed between releases; GPU acceleration irrelevant for CLI hooks. | `simple-statistics` for threshold-based classification logic |
| Python ML services (FastAPI + scikit-learn, etc.) | Introduces a second runtime, network hop, and service lifecycle management. Violates "local CLI tool" constraint. Adds 2-5 second latency in synchronous hook paths. | Pure JS libraries listed above |
| Cloud ML services (SageMaker, Vertex AI, Azure ML) | Violates budget constraint ($15/day), adds cloud dependency, introduces network latency incompatible with synchronous hook execution | Local statistical libraries |
| `node:sqlite` (Node.js built-in) | Experimental on Node.js 22 — requires `--experimental-sqlite` flag, API not stable, not suitable for production hook scripts that run on every tool use | `better-sqlite3@12.8.0` |
| LevelDB / `classic-level` | No SQL — range queries on time-series data (the core use case) require multi-step key scans. SQLite handles this in a single indexed query. | `better-sqlite3` |
| `lowdb` (JSON file database) | Loads entire dataset into memory on every read. Breaks on decision logs growing across weeks of sessions. | `better-sqlite3` |
| Neural networks for routing optimization | Small dataset (hundreds of sessions) makes neural nets statistically unreliable. Linear regression on structured features is more interpretable, more stable, and far simpler to debug. | `simple-statistics` + `ml-regression` |

---

## Stack Patterns by Variant

**If dataset stays small (under 5,000 decision records):**
- `simple-statistics` alone covers all needed analysis
- `simple-statistics.linearRegression()` handles trend detection
- Skip `ml-regression` to keep dependency count minimal

**If routing decisions need multi-feature correlation (task type + token count + model + latency combined):**
- Use `ml-regression` multivariate support or direct `ml-matrix` for multivariate linear regression
- SQLite `GROUP BY` + window functions handle feature extraction before regression input

**If you want cost anomaly alerting ("cost spiked 3x baseline"):**
- Z-score via `simple-statistics.zScore()` against a rolling 7-day mean
- No additional library needed — implement as a `lib/anomaly-detector.js` utility in the hooks directory

**If brain.js reaches a stable 2.x release:**
- Re-evaluate for routing decision classification (framing routing as a classification problem)
- Current beta state makes it unsuitable for a hook that fires on every tool use

---

## Version Compatibility — New Libraries

| Package | Version | Compatible With | Notes |
|---------|---------|-----------------|-------|
| `better-sqlite3` | 12.8.0 | Node.js v22.x | Node 22 support added May 2024. Requires `node-gyp` at install time; pre-built binaries usually available. |
| `simple-statistics` | 7.8.9 | Node.js v14+ | Pure JS, no native bindings, no compilation step. No compatibility concerns. |
| `ml-regression` | 6.3.0 | Node.js v12+ | Pure JS, no native bindings. Works with Node.js 22 without modification. |
| All three | — | `openai@6.33.0` | No dependency overlap. Safe to coexist in the same `package.json`. |

---

## Sources (ML Optimization Section)

- npm live version check (2026-04-03): `node -e "require('child_process').execSync('npm show <pkg> version')"` — confirmed `simple-statistics@7.8.9`, `ml-regression@6.3.0`, `better-sqlite3@12.8.0`, `brain.js@2.0.0-beta.24`
- [better-sqlite3 GitHub](https://github.com/WiseLibs/better-sqlite3) — synchronous API rationale, Node.js 22 support timeline
- [better-sqlite3 Discussion #1245](https://github.com/WiseLibs/better-sqlite3/discussions/1245) — production readiness vs `node:sqlite` — MEDIUM confidence
- [Node.js SQLite docs](https://nodejs.org/api/sqlite.html) — experimental flag requirement confirmed
- [simple-statistics npm](https://www.npmjs.com/package/simple-statistics) — zero dependencies confirmed
- [ml-regression npm](https://www.npmjs.com/package/ml-regression) — mljs ecosystem, pure JS confirmed
- [brain.js npm](https://www.npmjs.com/package/brain.js) — beta status, last publish ~April 2025 — HIGH confidence (direct npm registry)
- WebSearch: "Node.js machine learning libraries 2026 lightweight local inference" — candidate identification only, LOW-MEDIUM confidence

---
*Stack research additions for: ML-Driven Self-Optimization Milestone (Claude X Codex)*
*Researched: 2026-04-03*

---
---

# Stack Additions: Seraphim v3.0 — New Model Integrations and Plugin System

**Researched:** 2026-04-04
**Confidence:** HIGH for packages/versions (npm registry verified); HIGH for plugin manifest (official Claude Code docs); MEDIUM for Qwen model tag (ollama library confirmed, 27B does not exist — 30B is the correct target); HIGH for Perplexity/Gemini API patterns (official docs verified)

This section covers **new additions only** for Seraphim v3.0. The existing stack (Node.js v22.22.0, openai@6.33.0, codex CLI, MiniMax executor pattern) is carried forward unchanged.

---

## Scope: What This Section Covers

Five specific questions for v3.0:

1. Gemini 3.1 Pro and Gemini 3 Flash API integration
2. Qwen 3.5-27B local execution via ollama
3. Perplexity Sonar as a direct API executor (not just MCP)
4. Claude Code plugin.json manifest format
5. dispatch.js central routing pattern

---

## 1. Gemini API Integration

### SDK Decision: Use `@google/genai`, NOT `@google/generative-ai`

The older `@google/generative-ai` package (0.24.1) is **deprecated as of November 30, 2025**. It does not receive new features and does not support the Live API, Veo, or current model generations. The actively maintained replacement is `@google/genai`.

| Package | Version | Status | Decision |
|---------|---------|--------|----------|
| `@google/genai` | 1.48.0 | Active, current | USE THIS |
| `@google/generative-ai` | 0.24.1 | Deprecated Nov 2025 | DO NOT USE |

**Version verified:** npm registry live check 2026-04-04 → `@google/genai@1.48.0`

### Client Initialization

```javascript
const { GoogleGenAI } = require('@google/genai');

const ai = new GoogleGenAI({ apiKey: process.env.GEMINI_API_KEY });
```

Auth: `GEMINI_API_KEY` environment variable. Key must be configured before using gemini-exec.js. Currently not set on this machine — needs to be added.

### Text Generation (gemini-exec.js pattern)

```javascript
const response = await ai.models.generateContent({
  model: 'gemini-3.1-pro-preview',   // or 'gemini-3-flash-preview'
  contents: prompt
});
const text = response.text;
```

### Search Grounding (Discover phase — External track)

Enable Google Search as a tool for grounded web-aware responses with citations:

```javascript
const response = await ai.models.generateContent({
  model: 'gemini-3.1-pro-preview',
  contents: prompt,
  config: {
    tools: [{ googleSearch: {} }]
  }
});
```

Both `gemini-3.1-pro-preview` and `gemini-3-flash-preview` support the `googleSearch` tool. Older models used `google_search_retrieval` — do not use that form for current models.

### Function Calling (Internal discovery — tool simulation)

```javascript
const { FunctionCallingConfigMode } = require('@google/genai');

const response = await ai.models.generateContent({
  model: 'gemini-3.1-pro-preview',
  contents: prompt,
  config: {
    tools: [{ functionDeclarations: [yourFunctionDeclaration] }],
    toolConfig: {
      functionCallingConfig: {
        mode: FunctionCallingConfigMode.ANY
      }
    }
  }
});
```

### Streaming

```javascript
const stream = await ai.models.generateContentStream({
  model: 'gemini-3-flash-preview',
  contents: prompt
});
for await (const chunk of stream) {
  process.stdout.write(chunk.text);
}
```

### Model Names (confirmed from Gemini grounding docs, 2026-04-04)

| Model ID | Use In Seraphim | Why |
|----------|-----------------|-----|
| `gemini-3.1-pro-preview` | DISCOVER internal (Performance), ENVISION (Moderate), Opus fallback (Performance/Moderate) | 1M context, agentic multi-step planning, function calling |
| `gemini-3-flash-preview` | JUDGE (Performance), DISCOVER external (Budget/Frugal), CRUCIBLE verify (Moderate), Opus fallback (Frugal) | Fast, 90.4% GPQA Diamond, search grounding, low cost |

### Installation

```bash
npm install @google/genai@1.48.0
```

Install at `~/.claude/plugins/seraphim/` or wherever the plugin's `package.json` lives. The package is ~2MB, pure JS, no native bindings.

---

## 2. Qwen Local Execution via Ollama

### Model Tag Correction: 27B Does Not Exist

The design spec references "Qwen 3.5-27B" but this model size does not exist. The Qwen3 family on ollama has: 0.6B, 1.7B, 4B, 8B, 14B, 30B, 32B, 235B. The nearest equivalent to the intended 27B is the **30B** model.

**Recommended tag:** `qwen3:30b-a3b-q4_K_M`
- Size: 19GB (fits comfortably on RTX 3090 24GB VRAM)
- Quantization: Q4_K_M (good quality/size tradeoff, community standard for 4-bit)
- Context window: 256K tokens (more than enough; cap at 32K for consumer hardware as spec'd)
- Pull command: `ollama pull qwen3:30b-a3b-q4_K_M`

**Alternative for coding tasks:** `qwen3-coder:30b-a3b-q4_K_M` — same size, purpose-built for code generation. Consider for FORGE phase in Balanced/Budget profiles.

### Ollama HTTP API

Ollama exposes an OpenAI-compatible chat completions endpoint. This means `qwen-exec.js` can reuse the existing `openai` package (already installed at v6.33.0) with a baseURL swap — the same pattern used for MiniMax.

```javascript
const OpenAI = require('openai');

const client = new OpenAI({
  apiKey: 'ollama',            // any non-empty string; ollama ignores it
  baseURL: 'http://localhost:11434/v1'
});

const response = await client.chat.completions.create({
  model: 'qwen3:30b-a3b-q4_K_M',
  messages: [{ role: 'user', content: prompt }],
  temperature: 0.7,
  stream: false
});

const text = response.choices[0].message.content;
```

**Key parameters:**
- `baseURL`: `http://localhost:11434/v1` (bind to localhost per security rules)
- `model`: exact ollama model tag (e.g. `qwen3:30b-a3b-q4_K_M`)
- `stream`: set to `true` for streaming; ollama supports SSE
- No API key required; use any non-empty string

### Availability Check

```javascript
async function available() {
  try {
    const res = await fetch('http://localhost:11434/api/version', { signal: AbortSignal.timeout(2000) });
    return res.ok;
  } catch {
    return false;
  }
}
```

### Tool Simulation (Forge mode)

Qwen running via ollama cannot use MCP or Claude Code tools directly. The spec's approach — structured JSON output interpreted by the wrapper — is correct:

Prompt Qwen to output actions in this schema:
```json
{"action": "write", "path": "src/foo.js", "content": "..."}
{"action": "run", "command": "npm test"}
{"action": "read", "path": "package.json"}
```

The `qwen-exec.js` wrapper parses this output and executes the actions. This is a well-established pattern for local models without native tool support.

### Timeout

Use 120s as specified. Qwen 30B Q4 on RTX 3090 generates approximately 15-25 tokens/second. A 2,000 token response takes ~80-130 seconds — 120s is appropriate.

### No New npm Dependencies

`qwen-exec.js` reuses the existing `openai@6.33.0` package via baseURL swap. Zero new installs needed for Qwen support.

---

## 3. Perplexity Sonar as Direct API Executor

### API Pattern: OpenAI SDK baseURL Swap (same as MiniMax)

Perplexity's Sonar API is OpenAI-compatible. `perplexity-exec.js` reuses the existing `openai@6.33.0` package exactly as `minimax-exec.js` does.

```javascript
const OpenAI = require('openai');

const client = new OpenAI({
  apiKey: process.env.PERPLEXITY_API_KEY,
  baseURL: 'https://api.perplexity.ai'
});

const response = await client.chat.completions.create({
  model: 'sonar-pro',
  messages: [
    { role: 'system', content: 'Be precise and cite sources.' },
    { role: 'user', content: prompt }
  ],
  stream: false
});

const text = response.choices[0].message.content;
// Citations are in response.citations (Perplexity-specific extension)
const citations = response.citations || [];
```

### Model IDs (confirmed from official Perplexity docs, 2026-04-04)

| Model ID | Use In Seraphim | Why |
|----------|-----------------|-----|
| `sonar-pro` | DISCOVER external (Performance, Balanced, Moderate) | Web-grounded, complex multi-step queries, 2x citations vs sonar |
| `sonar` | DISCOVER external (lightweight fallback) | Cheaper, still grounded, fast |
| `sonar-reasoning-pro` | When reasoning + search needed | DeepSeek R1 backbone + CoT + web search; expensive |
| `sonar-deep-research` | Deep research tasks only | Long-form exhaustive reports; async; use sparingly |

**Recommended default:** `sonar-pro` for DISCOVER external. It provides more citations than `sonar` and handles follow-up queries better.

### Base URL

`https://api.perplexity.ai` — no `/v1` suffix (unlike OpenAI and MiniMax which use `/v1`).

### Citations in Response

Perplexity returns citations as a `citations` array on the response object (outside the standard OpenAI schema). Capture them separately from the message content:

```javascript
const result = {
  text: response.choices[0].message.content,
  citations: response.citations || [],
  model: response.model
};
```

Write citations to the phase output file alongside the research text for traceability.

### Environment Variable

`PERPLEXITY_API_KEY` — not currently set as an env var on this machine (only available via MCP). Needs to be added to the environment for the direct API executor to work.

### No New npm Dependencies

Zero additional packages needed. The existing `openai@6.33.0` handles the baseURL swap.

---

## 4. Claude Code Plugin Manifest

### Directory Structure

The plugin manifest lives at `.claude-plugin/plugin.json` inside the plugin directory. All other components (commands, agents, hooks, executors, tools) live at the **plugin root**, not inside `.claude-plugin/`.

**Correct layout for Seraphim:**

```
~/.claude/plugins/seraphim/
├── .claude-plugin/
│   └── plugin.json          # Manifest — ONLY file in this directory
├── commands/                # /seraphim:discover, etc. (.md files)
├── agents/                  # seraphim-envision.md, etc.
├── hooks/
│   └── hooks.json           # SessionStart, PostToolUse config
├── executors/               # gemini-exec.js, qwen-exec.js, etc.
├── tools/                   # websearch.sh, checkpoint.js, etc.
├── lib/                     # pricing.js, config.js, phase-state.js
└── config/                  # profiles.json, models.json
```

The manifest is **optional** — Claude Code auto-discovers components without it. Include it to get namespacing, userConfig prompts, and precise path declarations.

### plugin.json Schema

```json
{
  "name": "seraphim",
  "version": "3.0.0",
  "description": "Six-phase multi-model creative pipeline. Nine models, five profiles.",
  "author": {
    "name": "Dragos"
  },
  "commands": "./commands/",
  "agents": "./agents/",
  "hooks": "./hooks/hooks.json",
  "userConfig": {
    "gemini_api_key": {
      "description": "Google AI Studio API key for Gemini models",
      "sensitive": true
    },
    "perplexity_api_key": {
      "description": "Perplexity API key for Sonar web search",
      "sensitive": true
    }
  }
}
```

**Key decisions:**
- `name: "seraphim"` — commands become `/seraphim:discover`, agents become `seraphim:seraphim-envision`, etc.
- Sensitive `userConfig` values go to system keychain, not `settings.json`
- `hooks` points to a JSON file rather than inlining — keeps the manifest readable
- Executors, tools, and lib are NOT declared in the manifest — they're Node.js modules required by the hooks and commands, not Claude Code components

### hooks.json Pattern

```json
{
  "hooks": {
    "SessionStart": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "${CLAUDE_PLUGIN_ROOT}/hooks/session-start.js"
          }
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Write|Edit|MultiEdit",
        "hooks": [
          {
            "type": "command",
            "command": "${CLAUDE_PLUGIN_ROOT}/hooks/token-logger.js"
          }
        ]
      }
    ]
  }
}
```

Use `${CLAUDE_PLUGIN_ROOT}` for all paths inside hook commands — this variable resolves to the plugin's installation directory regardless of where it is installed.

### Persistent Data Directory

Use `${CLAUDE_PLUGIN_DATA}` for state that must survive plugin updates (token logs, decisions.jsonl, per-project configs). This resolves to `~/.claude/plugins/data/seraphim/`.

The per-project `.seraphim/` directory is separate from this — it lives in each project directory and is managed by `lib/config.js`.

### Plugin Installation

```bash
# Install from local directory for development
claude --plugin-dir ~/.claude/plugins/seraphim

# Or install as user-scope plugin (persists across sessions)
# (requires marketplace entry for permanent install)
```

For development, use `--plugin-dir` flag. For production deployment, the plugin will need a marketplace entry or manual `settings.json` registration.

### Validation

```bash
claude plugin validate
# or from within a session:
# /plugin validate
```

Run this after creating/modifying `plugin.json` to catch schema errors before they cause silent failures.

---

## 5. dispatch.js Central Routing Pattern

### Design Principle

`dispatch.js` is the single indirection point between phase logic and model executors. It reads config, resolves the phase-to-model mapping, loads the correct executor module, and calls it. No phase command should `require()` a specific executor directly.

### Recommended Implementation Pattern

```javascript
// executors/dispatch.js
'use strict';

const path = require('path');
const configLib = require('../lib/config');
const profilesData = require('../config/profiles.json');

const EXECUTOR_MAP = {
  'claude-opus-4.6':       null,          // native session — Opus calls itself
  'claude-sonnet-4.6':     null,          // subagent — handled by agent runner
  'claude-haiku-4.5':      null,          // subagent — handled by agent runner
  'codex-gpt-5.4':         './codex-exec',
  'minimax-m2.7':          './minimax-exec',
  'gemini-3.1-pro':        './gemini-exec',
  'gemini-3-flash':        './gemini-exec',  // same executor, different model param
  'qwen-3.5-27b':          './qwen-exec',    // actually 30b — legacy name kept in config
  'perplexity-sonar':      './perplexity-exec'
};

/**
 * Resolve which model handles a given phase, applying profile + overrides.
 * @param {string} phase - e.g. 'discover_external', 'judge', 'forge'
 * @param {object} projectConfig - loaded from .seraphim/config.json
 * @returns {string} - model key from EXECUTOR_MAP
 */
function resolveModel(phase, projectConfig) {
  // 1. Check per-phase override
  if (projectConfig.overrides && projectConfig.overrides[phase]) {
    return projectConfig.overrides[phase];
  }

  // 2. Apply profile defaults
  const profile = profilesData[projectConfig.profile || 'moderate'];
  const model = profile.phases[phase];

  // 3. Apply opus_enabled toggle
  if (!projectConfig.opus_enabled && model && model.startsWith('claude-opus')) {
    return profile.opus_fallback;
  }

  return model;
}

/**
 * Dispatch a prompt to the correct executor for a given phase.
 * @param {string} phase - phase key
 * @param {string} prompt - the prompt to send
 * @param {object} opts - { timeout, temperature, stream, ... }
 * @returns {Promise<{success, output, tokens, cost, error}>}
 */
async function dispatch(phase, prompt, opts = {}) {
  const projectConfig = configLib.load();
  const modelKey = resolveModel(phase, projectConfig);

  const executorPath = EXECUTOR_MAP[modelKey];
  if (!executorPath) {
    // Claude subagents or native session — caller handles directly
    return { success: false, error: `Model ${modelKey} requires native session or subagent, not dispatch` };
  }

  const executor = require(path.join(__dirname, executorPath));

  // Check availability before attempting
  if (executor.available && !(await executor.available())) {
    return { success: false, error: `Executor for ${modelKey} is not available` };
  }

  return executor.execute(prompt, { model: modelKey, ...opts });
}

module.exports = { dispatch, resolveModel };
```

### Executor Interface Contract

Every executor module must export this interface:

```javascript
module.exports = {
  // Run a prompt, return structured result
  async execute(prompt, opts) {
    // opts: { model, temperature, timeout, maxTokens, ... }
    return {
      success: Boolean,
      output: String,          // the model's response text
      tokens: { input: Number, output: Number },
      cost: Number,            // USD
      error: String | null     // null on success
    };
  },

  // Streaming variant (optional — implement only where supported)
  async stream(prompt, opts) {
    // returns AsyncIterable of string chunks
  },

  // Can this executor run right now?
  async available() {
    return Boolean;
  }
};
```

All three existing executors (codex-exec.js, minimax-exec.js) and all three new executors (gemini-exec.js, qwen-exec.js, perplexity-exec.js) implement this interface.

### Gemini Executor Model Parameter

`gemini-exec.js` handles both `gemini-3.1-pro` and `gemini-3-flash` since the SDK is the same. The `model` key from opts maps to the actual model ID:

```javascript
const MODEL_IDS = {
  'gemini-3.1-pro':  'gemini-3.1-pro-preview',
  'gemini-3-flash':  'gemini-3-flash-preview'
};
```

---

## Installation Summary

All new packages for v3.0:

```bash
# Install in the plugin directory
cd ~/.claude/plugins/seraphim

# The only new npm package needed for v3.0
npm install @google/genai@1.48.0

# No additional packages needed for:
# - Qwen (reuses openai@6.33.0 via baseURL swap)
# - Perplexity (reuses openai@6.33.0 via baseURL swap)
# - dispatch.js (pure Node.js, no deps)
# - plugin.json (Claude Code reads it natively)
```

Ollama installation (when RTX 3090 arrives):
```bash
curl -fsSL https://ollama.com/install.sh | sh
ollama pull qwen3:30b-a3b-q4_K_M
```

---

## Alternatives Considered

| Category | Recommended | Alternative | Why Not |
|----------|-------------|-------------|---------|
| Gemini SDK | `@google/genai@1.48.0` | `@google/generative-ai@0.24.1` | Deprecated Nov 2025; no new features; no current model support |
| Qwen access | ollama HTTP (OpenAI compat) | ollama native `/api/generate` endpoint | OpenAI-compat endpoint enables reusing existing `openai` package; `/api/generate` requires a custom HTTP client |
| Qwen access | ollama HTTP (OpenAI compat) | `ollama` npm package (unofficial) | No official npm package; the HTTP API is the official interface |
| Perplexity access | OpenAI SDK baseURL swap | `perplexity-sdk` npm (unofficial) | Unofficial, minimal maintenance; OpenAI SDK is already installed and the baseURL pattern is proven |
| Perplexity access | Direct `sonar-pro` | Perplexity MCP only | MCP is unavailable to subagents and non-Claude executors; direct API is necessary for consistent access across all dispatch paths |
| dispatch.js | Lazy `require()` per call | Pre-load all executors at startup | Pre-loading causes ollama connectivity check failures on startup when GPU is absent; lazy loading means only the requested executor is checked |

---

## What NOT to Use

| Avoid | Why | Use Instead |
|-------|-----|-------------|
| `@google/generative-ai` | Deprecated Nov 30, 2025; missing current model support and Live API features | `@google/genai@1.48.0` |
| Vertex AI SDK | Requires GCP project, service account, and project-scoped auth; this is a personal tool using Google AI Studio | `@google/genai` with `GEMINI_API_KEY` (AI Studio key) |
| `ollama` npm package (unofficial, e.g. `npm:ollama`) | Not the official client; the Ollama project's official interface is the HTTP API | Direct HTTP via `openai` SDK with `baseURL: 'http://localhost:11434/v1'` |
| Perplexity MCP as the only access path | MCP tools are only available to Claude in the native session; executors, subagents, and hooks cannot use MCP tools | `perplexity-exec.js` using direct API |
| `sonar-reasoning-pro` as default | 3-5x more expensive than `sonar-pro`; CoT reasoning is not needed for web research | `sonar-pro` as default; `sonar-reasoning-pro` as optional override |
| Global npm install for plugin packages | Plugin dependencies installed globally conflict across projects and survive plugin uninstall | Install in `~/.claude/plugins/seraphim/node_modules/` using `${CLAUDE_PLUGIN_DATA}` pattern from hooks.json |

---

## Stack Patterns by Variant

**If RTX 3090 is not yet installed (current state):**
- Balanced and Budget profiles are unavailable; dispatch.js should return `{ success: false, error: 'Qwen requires GPU — use Moderate or Frugal profile' }` from `available()`
- Performance, Moderate, and Frugal profiles work fully with cloud APIs

**If GEMINI_API_KEY is not set:**
- `gemini-exec.js` `available()` returns false
- dispatch.js falls back per profile config (e.g., Performance falls back to Opus for both Gemini phases)

**If PERPLEXITY_API_KEY is not set:**
- `perplexity-exec.js` `available()` returns false
- Discover external falls back to Gemini 3 Flash (which has Google Search grounding)

**If running Frugal profile with no Qwen:**
- All Frugal phases use cloud models — no GPU dependency

---

## Version Compatibility — v3.0 Additions

| Package | Version | Node.js Requirement | Notes |
|---------|---------|---------------------|-------|
| `@google/genai` | 1.48.0 | Node.js v18+ (ESM + CJS) | CJS require() works; pure JS, no native bindings |
| `openai` (Qwen baseURL) | 6.33.0 (existing) | Node.js v18+ | Reused; no version conflict |
| `openai` (Perplexity baseURL) | 6.33.0 (existing) | Node.js v18+ | Reused; no version conflict |
| ollama HTTP API | server v0.x | Any HTTP client | Not an npm package; version depends on installed ollama binary |

---

## Sources (v3.0 Section)

- `@google/genai` version: npm registry live check 2026-04-04 → `1.48.0` — HIGH confidence
- `@google/generative-ai` version: npm registry live check 2026-04-04 → `0.24.1` (deprecated) — HIGH confidence
- Gemini SDK deprecation: https://ai.google.dev/gemini-api/docs/sdks — "legacy libraries deprecated November 30, 2025" — HIGH confidence
- Gemini search grounding API: https://ai.google.dev/gemini-api/docs/grounding — `tools: [{ googleSearch: {} }]` pattern — HIGH confidence
- Gemini `@google/genai` README: https://github.com/googleapis/js-genai — `ai.models.generateContent()` pattern — HIGH confidence
- Ollama OpenAI compatibility: https://ollama.com/blog/openai-compatibility — `http://localhost:11434/v1` endpoint — HIGH confidence
- Qwen3 30B Q4 tag: https://ollama.com/library/qwen3/tags — `qwen3:30b-a3b-q4_K_M`, 19GB — HIGH confidence
- Qwen3 27B non-existence: https://ollama.com/library/qwen3 — only 0.6B, 1.7B, 4B, 8B, 14B, 30B, 32B, 235B listed — HIGH confidence
- Perplexity API base URL: https://docs.perplexity.ai/docs/sonar/quickstart — `https://api.perplexity.ai` — HIGH confidence
- Perplexity model IDs: WebSearch + https://docs.perplexity.ai/models/model-cards — `sonar`, `sonar-pro`, `sonar-reasoning-pro`, `sonar-deep-research` — HIGH confidence
- Perplexity OpenAI compat: https://docs.perplexity.ai/docs/agent-api/openai-compatibility — confirmed — HIGH confidence
- Claude Code plugin schema: https://code.claude.com/docs/en/plugins-reference — complete manifest schema, `.claude-plugin/plugin.json` location, `${CLAUDE_PLUGIN_ROOT}` variable — HIGH confidence
- `openai@6.33.0` installed version: `node -e "require('/home/alucard/.npm-global/lib/node_modules/openai/package.json').version"` → `6.33.0` — CONFIRMED live

---
*Stack research additions for: Seraphim v3.0 Six-Phase Multi-Model Pipeline*
*Researched: 2026-04-04*
