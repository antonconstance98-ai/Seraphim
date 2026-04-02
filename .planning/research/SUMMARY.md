# Project Research Summary

**Project:** Claude X Codex — v1.1 Global Metrics Dashboard
**Domain:** Local HTML metrics dashboard for multi-project AI model usage, cost, and review-quality tracking
**Researched:** 2026-04-02
**Confidence:** HIGH

## Executive Summary

This milestone adds a global, cross-project metrics dashboard to the fully operational v1.0 Claude X Codex hook system. The v1.0 foundation (Codex CLI routing, cross-model review gate, per-project JSONL token logging) is unchanged. The v1.1 work is entirely additive: a new `SessionStart` hook aggregates per-project `token-log.jsonl` files into a global store and generates a self-contained HTML dashboard. The entire feature requires zero new npm packages — Node.js v22 built-ins handle file discovery, the HTML is generated with ES6 template literals matching the existing `codex-cost-reporter.js` pattern, and Chart.js is downloaded once and cached as a sidecar file.

The recommended approach is a two-component architecture: `codex-global-aggregator.js` handles project discovery and JSONL merging (with deduplication, incremental reads, and a `project-index.json` manifest), and `codex-dashboard-generator.js` transforms the global log into a self-contained HTML file. Both run in the same `SessionStart` hook invocation via `require()` composition, keeping overhead under the 30-second hook timeout. The output is a single `~/.claude/dashboard/dashboard.html` that opens directly in a browser from `file://`. All data is serialized as a `const DASHBOARD_DATA = {...}` block at generation time, satisfying the offline / no-server constraint. Existing per-project hooks and JSONL files are untouched.

The top risks for this milestone require day-one design decisions and cannot be retrofitted later: (1) the aggregator scan must be bounded to specific root directories with `-maxdepth` limits or it will breach the 30-second hook timeout; (2) the dashboard HTML must be written via write-then-`renameSync` or concurrent session starts will corrupt it; (3) records with `session_id: null` (confirmed 25% of live data from `codex-multi-round-reviewer.js`) must be explicitly handled as "unattributed" rather than silently dropped; and (4) pricing constants must be centralized into a single shared module before the dashboard consumes them, or unknown model names will silently return `$0` and inflate the savings metric to 100%.

---

## Key Findings

### Recommended Stack

No new npm packages are required for the v1.1 dashboard. All new capabilities use Node.js v22 built-ins or a single locally-cached static file. This is consistent with the existing hook codebase pattern and eliminates any `npm install` step.

**Core technologies:**

- **Node.js v22.22.0 (installed)**: All hook scripts, aggregation logic, HTML generation — matched to existing codebase; `fs/promises.glob()` async iterator replaces the `glob` npm package entirely on v22
- **`fs/promises` built-in (`glob`, `readFile`, `appendFile`, `writeFile`, `rename`)**: Project discovery and JSONL I/O — no external dependency; `glob()` returns an `AsyncIterator` (must use `for-await-of`, not `.then()` — verified live)
- **`fetch()` built-in (Node.js v18+)**: One-time Chart.js download — available as global on v22.22.0, no import needed
- **Chart.js 4.5.1 UMD build (204 KB, cached once at `~/.claude/dashboard/assets/`)**: Line + bar charts — downloaded once from jsDelivr CDN, referenced via relative `<script src="./assets/chart.umd.min.js">` sidecar (not re-inlined on every generation)
- **ES6 template literals**: HTML generation — same pattern as existing `codex-cost-reporter.js`; no template engine needed

**Critical API note:** `fs.glob()` on Node.js v22 returns an `AsyncIterator`, not a Promise. Using `.then()` on it throws `TypeError: glob(...).then is not a function`. This was confirmed by live testing. Use `for await...of` exclusively.

**Do not use:** `glob`/`fast-glob` npm packages (superseded by built-in), EJS/Mustache/Eta (template engines for Express apps), React/Vue/Svelte (require build step), D3.js (500 KB, overkill), CDN `<script>` tags (dashboard must work offline), `chartjs-node-canvas` (server-side PNG rendering, no interactivity), any web framework (Express, Fastify — no server needed or wanted).

### Expected Features

The dashboard must deliver visible value beyond the existing per-project Markdown session reports. Every visual feature depends on the global aggregator completing successfully — it is the root dependency for all rendering.

**Must have (table stakes — v1.1 launch):**

- **Global cost summary card** — total actual cost, Opus-equivalent baseline, savings %, total calls, global catch rate; the primary reason to build this
- **Per-project breakdown table** — project, calls, actual cost, savings $, savings %, catch rate, cache efficiency %; answers "which project drove which cost"
- **Time trend line chart (daily, last 30 days)** — cost trajectory impossible to see in tabular Markdown reports
- **Session history table** — cross-project view of last 20 sessions; groups by non-null `session_id`; unattributed row for null-session records
- **BLOCK issue log** — timestamps, project, task type, block_summary for all `verdict === 'BLOCK'` records; "proof of value" log showing what Codex caught
- **Self-contained HTML (no server, no `fetch()` at render time)** — all data inlined as `const DASHBOARD_DATA`; opens from `file://`
- **Auto-regenerate on `SessionStart`** — second hook alongside existing `codex-cost-reporter.js`; skips regeneration if no new records (sentinel file guard)
- **Last-updated footer** — ISO timestamp written at generation time; prevents acting on stale data

**Should have (v1.x follow-on patch):**

- Weekly trend toggle — client-side JS regroups same daily data into ISO week buckets; no new data structures
- Project comparison horizontal bar chart — per-project cost and savings side-by-side
- Model split chart (pie or bar) — GPT-5.4 vs GPT-5.4-mini token and cost share

**Defer (v2+):**

- Date range filter (client-side JS) — last 7/30/90 days selector
- Task-type drill-down — click project row to see task breakdown in a modal
- Export to CSV — download aggregated data for external analysis

**Anti-features (do not build):**

- Real-time auto-refresh (`setInterval` polling) — JSONL is append-only; `SessionStart` regeneration is sufficient
- Server process (Express, Python HTTP) — violates `file://` constraint; every server is a process to manage
- SQLite or any database — JSONL aggregation at generation time takes under 100 ms at current scale
- Cross-machine aggregation — explicitly out of scope; single-machine local files only

### Architecture Approach

The architecture adds two new hook scripts to `~/.claude/hooks/` and a new `~/.claude/dashboard/` directory. All existing components (`codex-token-logger.js`, `codex-cost-reporter.js`, per-project JSONL files, all other hooks) are untouched. The aggregator and generator are composed via `require()` within the same Node.js process, avoiding a second process startup overhead. The global store is an append-only `global.jsonl` that preserves historical records permanently even if per-project source files are deleted or rotated.

**Major components:**

1. **`codex-global-aggregator.js`** (new `SessionStart` hook) — discovers all projects via bounded `find` command across configurable roots, deduplicates records against `global.jsonl` using `session_id + timestamp` composite key, appends new records enriched with `project_path` and `project_name`, writes `project-index.json` manifest and `last-run.json` idempotency guard, calls dashboard generator via `require()`
2. **`codex-dashboard-generator.js`** (new generator module, called via `require()`) — reads `global.jsonl`, computes per-project totals / daily time series / session history / global totals, writes self-contained `dashboard.html` via write-then-`renameSync`; pure transform with no side effects outside `~/.claude/dashboard/`
3. **`~/.claude/dashboard/`** (new auto-created directory) — holds `global.jsonl` (append-only global record store), `project-index.json` (discovery manifest), `dashboard.html` (output opened in browser), `last-run.json` (idempotency guard), `cache.json` (incremental read state tracking mtime + byte offset per file), `assets/chart.umd.min.js` (Chart.js sidecar)
4. **`~/.claude/settings.json`** (one modification) — adds `codex-global-aggregator.js` to `SessionStart` hook array after `codex-cost-reporter.js` with `timeout: 30`

**Key architectural patterns:**

- **Append-only global log with in-memory dedup Set** — never rewrite `global.jsonl`; load existing `session_id+timestamp` keys into a `Set` at startup, append only truly new records; preserves history even when source files are deleted
- **Generator as pure transform** — aggregator owns discovery and merging; generator owns computation and rendering; each testable in isolation via `node script.js`
- **Write-then-rename for all shared output files** — write to `dashboard.html.tmp.{pid}`, then `fs.renameSync()` into place; `rename(2)` is atomic on Linux same-filesystem; protects against concurrent session start corruption
- **mtime-gated incremental reads** — `fs.statSync` each log file; if `mtime` and `size` unchanged vs `cache.json`, skip re-parsing; seek to last byte offset for new-only lines; keeps hook overhead under 5 ms on repeat sessions

### Critical Pitfalls

**v1.1-specific pitfalls (new for this milestone):**

1. **Unbounded `SessionStart` filesystem scan (v1.1 Pitfall A)** — scanning `~/` recursively without depth limits hits `node_modules`, `.git`, and cloud-sync directories; timed experiment shows non-linear scaling that can exceed the 30-second hook timeout. Prevention: hardcode scan to specific roots (`~/projects`, `~/agent`, `~/gsd-workspaces`, `/mnt/hdd`) with `-maxdepth 5`; set `timeout: 30` on the hook in `settings.json`. Must be designed in before any aggregation logic is written.

2. **Concurrent dashboard HTML write corruption (v1.1 Pitfall B)** — two Claude Code sessions opening simultaneously both call `writeFileSync` on the same `dashboard.html`; `writeFileSync` does truncate-then-write (not atomic), producing an empty or torn file. Prevention: always use write-to-temp-then-`fs.renameSync()`. One line of code; must be implemented from day one, not retrofitted.

3. **Null `session_id` silently drops 25% of records from session history (v1.1 Pitfall C)** — records from `codex-multi-round-reviewer.js` have `session_id: null` because it is called as a shared library without receiving `session_id` from the calling hook. Confirmed: 3 of 12 live records in `Claude_X_Codex/.planning/token-log.jsonl` have `session_id: null`. Prevention: fix the root cause (pass `session_id` from `codex-plan-reviewer.js` and `codex-superpowers-plan-reviewer.js` into the shared library), or label null-session records as "Unattributed" in session history. Never silently drop them.

4. **Hardcoded pricing returns `$0` for unknown models (v1.1 Pitfall F)** — existing `codex-cost-reporter.js` returns `0` when a model name has no pricing entry, making all calls from that model appear free and inflating savings to 100%. Prevention: centralize all pricing in a single `pricing.js` module shared by all hooks and the dashboard generator; use `null` (not `0`) for unknown models; surface a visible warning in the dashboard. Must be done in Phase 1 before the dashboard consumes pricing data.

5. **Re-parsing all JSONL records on every session start (v1.1 Pitfall E)** — acceptable at 11 records now (under 1 ms); approaches 200 ms at 100,000 records (18 months of active use). Prevention: implement mtime-gated incremental reads in Phase 1 using `fs.statSync` + `cache.json`; seek to last byte offset for new-only lines. Harder to retrofit than to design in.

**Carry-forward from v1.0 (remain relevant for all hook work):**

6. **Codex CLI silent background process hang (Pitfall 1)** — open bug in CLI 0.114.0+; all `codex exec` hook calls must use `timeout 300` wrapper; use `--approval-policy untrusted` to prevent persistent background subprocesses
7. **API key exfiltration via hook scripts (Pitfall 2, CVE-2025-59536)** — patched in Claude Code 2.0.65+; enable `CLAUDE_CODE_SUBPROCESS_ENV_SCRUB=1`; never echo or log environment variables in hook scripts
8. **Cost runaway from routing failure (Pitfall 4)** — hard daily spend cap at $10 (buffer before $15 ceiling); routing must fail closed (prompt user), not fall back to Opus automatically

---

## Implications for Roadmap

The build order is driven by data dependency: the aggregator must produce a verified `global.jsonl` before the dashboard generator can be meaningfully tested against real data. Phase 1 installs the data pipeline and validates correctness in isolation. Phase 2 builds the rendering layer on top of confirmed data. Phase 3 adds Chart.js charts and wires the hook for full production use.

### Phase 1: Data Pipeline — Global Aggregator

**Rationale:** The aggregator is the root dependency for all dashboard features. Every visual feature requires `global.jsonl` to exist and be accurate. Implementing and validating the aggregator in isolation means the rendering layer in Phase 2 can be tested against real, correct data from day one. This phase also contains four "must be designed in from the start" decisions (scan bounds, dedup key, null session_id handling, incremental reads with mtime guard) that cannot be cleanly retrofitted after Phase 2 is built.

**Delivers:** `codex-global-aggregator.js` producing a correct, deduplicated `global.jsonl` from all known projects; `project-index.json` discovery manifest; `last-run.json` idempotency guard; `cache.json` incremental read state; centralized `pricing.js` module.

**Addresses:** Global aggregation (prerequisite for all dashboard features), session grouping logic (null `session_id` decision), project discovery (scan roots + exclusions), pricing accuracy.

**Avoids:** Pitfall A (bounded scan with explicit roots and `-maxdepth`), Pitfall C (null session_id explicitly handled as "Unattributed"), Pitfall E (incremental reads from the start), Pitfall F (centralized pricing module before dashboard consumes it).

**Verification:** Run `node ~/.claude/hooks/codex-global-aggregator.js` standalone; confirm `global.jsonl` contains correct records from both `Claude_X_Codex` and `The-Crucible`; confirm null-session records appear as "Unattributed" not missing; confirm second consecutive run completes in under 5 ms with no new records added.

### Phase 2: Dashboard Generator — HTML Output

**Rationale:** Phase 2 consumes the verified `global.jsonl` from Phase 1. Building rendering after data validation means chart calculations and table values can be checked against known-correct totals. The write-then-rename pattern and the sentinel file guard (skip regeneration when no new data) must be implemented in this phase before the hook is registered in Phase 3.

**Delivers:** `codex-dashboard-generator.js` producing `~/.claude/dashboard/dashboard.html` with: global summary section, per-project breakdown table, session history table (with "Unattributed" row), BLOCK issue log, and last-updated footer.

**Uses:** ES6 template literals (existing codebase pattern), Chart.js 4.5.1 sidecar (`assets/chart.umd.min.js` downloaded once, referenced via relative path), `const DASHBOARD_DATA = JSON.stringify(aggregated)` data inlining for `file://` compatibility.

**Implements:** Generator-as-pure-transform pattern; write-then-`renameSync` for concurrent write safety; sentinel `last-generated.json` to skip regeneration when no new records.

**Avoids:** Pitfall B (write-then-rename implemented from day one), Pitfall D (Chart.js sidecar written once, not 204 KB re-inlined per session), Pitfall G (sentinel file prevents unnecessary regeneration).

**Verification:** Run `node ~/.claude/hooks/codex-dashboard-generator.js` standalone; open `~/.claude/dashboard/dashboard.html` from `file://` in browser; confirm all tables render correctly; verify charts render (not blocked by browser security); manually test concurrent write safety by starting two sessions simultaneously and confirming no torn file.

### Phase 3: Hook Integration and Chart Visualizations

**Rationale:** Register the hook only after the standalone generator is verified correct. Adding Chart.js time-series charts last means data structures are stable and chart rendering issues will not contaminate data pipeline debugging.

**Delivers:** `SessionStart` hook registration in `~/.claude/settings.json` (after `codex-cost-reporter.js`, `timeout: 30`); time trend line chart (daily cost/savings over last 30 days); end-to-end flow from session open to dashboard update.

**Uses:** `SessionStart` hook array; Chart.js 4.5.1 `new Chart(canvas, { type: 'line', ... })` API; `require('./codex-dashboard-generator')` composition pattern.

**Avoids:** Pitfall A (verified hook completes within 30-second budget with real multi-project scan).

**Verification:** Open a Claude Code session; confirm `global.jsonl` updated; confirm `dashboard.html` regenerated; confirm session open delay is under 2 seconds; open dashboard from `file://` and confirm line chart renders with real data.

### Phase Ordering Rationale

- **Data before rendering:** The aggregator must produce a correct `global.jsonl` before any rendering code is written. Testing the generator against incomplete data produces misleading results that are hard to distinguish from rendering bugs.
- **Standalone before hooked:** Both new scripts should be tested by running them directly (`node script.js`) before they are registered as `SessionStart` hooks. A broken hook blocks every session open; a broken standalone script is trivially debugged.
- **Charts last:** Chart rendering is a display concern. Correctness of the underlying data (dedup, null session handling, pricing) matters more and is harder to debug once charts are in the picture.
- **Pricing centralized in Phase 1:** The pricing module must exist before the dashboard consumes it. Building the dashboard first and adding centralized pricing later means two sources of truth exist temporarily — a known tech debt pattern that causes silent drift.

### Research Flags

All three phases follow well-documented patterns already established in this codebase. No `/gsd:research-phase` is needed before planning any phase.

**Phases with standard patterns (skip research-phase):**

- **Phase 1 (Aggregator):** All patterns are present in the live codebase. `codex-cost-reporter.js` is the canonical reference for JSONL reading and report generation. The `find` command pattern and `Set`-based dedup are standard Node.js. The `fs.glob()` async iterator API is confirmed working on this machine.
- **Phase 2 (Generator):** ES6 template literal HTML generation is an established pattern in this codebase. Chart.js API is well-documented; UMD build confirmed at v4.5.1 on this machine.
- **Phase 3 (Hook Integration):** Hook registration format is live in `~/.claude/settings.json`. Pattern is identical to existing `codex-cost-reporter.js` registration. SessionStart hook execution order confirmed by live `settings.json` inspection.

---

## Confidence Assessment

| Area | Confidence | Notes |
|------|------------|-------|
| Stack | HIGH | All technologies tested live on this machine. `fs.glob()` confirmed working with correct `AsyncIterator` behavior on v22.22.0. Chart.js 4.5.1 download and 204 KB size verified. Existing JSONL schema confirmed from live files. |
| Features | HIGH | Feature set derived from live data schema (verified) and direct comparison against existing per-project Markdown reports. MVP scope is conservative and deliverable in three phases. |
| Architecture | HIGH | Architecture based on live source code inspection of `codex-cost-reporter.js`, `codex-token-logger.js`, and `settings.json`. Two live `token-log.jsonl` files across two projects confirm cross-project aggregation is feasible. Dedup key design verified against live data showing null `session_id` records. |
| Pitfalls | HIGH | v1.1 pitfalls backed by timed live experiments (filesystem scan timing, JSONL parse timing at 100K-record projection), live data analysis (25% null `session_id` counted from actual records), and Linux `rename(2)` POSIX atomicity specification. General carry-forward pitfalls backed by GitHub issues with version numbers and CVE disclosures. |

**Overall confidence:** HIGH

### Gaps to Address

- **Chart.js `<canvas>` rendering from `file://` in restricted browser security modes (v1.1 Pitfall D):** Chart.js requires `<canvas>` which some browsers block when loading from `file://`. Research recommends SVG path charts as the universal fallback (50-100 lines of SVG math). The sidecar approach avoids the 204 KB per-generation issue, but if canvas rendering fails in Phase 2 verification, SVG charts are the fallback path. Validate Chart.js from `file://` in Phase 2 before committing to the approach.

- **`session_id` propagation fix scope:** Pitfall C recommends passing `session_id` from `codex-plan-reviewer.js` and `codex-superpowers-plan-reviewer.js` into `codex-multi-round-reviewer.js`. This modifies two existing v1.0 hook files. The scope is small (1-2 lines each), but the shared library's function signature must be confirmed before Phase 1 is marked complete. If the fix cannot be made cleanly, the "Unattributed" labeling approach is the acceptable alternative.

- **`mtime` reliability on `/mnt/hdd` (Windows NTFS filesystem mount):** The incremental read guard relies on `fs.statSync` mtime precision. NTFS mounted via WSL/FUSE has 100 ns mtime resolution, which is sufficient. Validate with a live test if any project is stored under `/mnt/hdd` and the cache is not updating correctly.

---

## Sources

### Primary (HIGH confidence)

- Live source: `/home/alucard/.claude/hooks/codex-cost-reporter.js` — SessionStart hook pattern, JSONL reading, report generation, pricing constants
- Live source: `/home/alucard/.claude/hooks/codex-token-logger.js` — JSONL record schema, `[CODEX_RESULT]` marker protocol
- Live source: `/home/alucard/.claude/settings.json` — hook registration format, timeout values, SessionStart array structure
- Live data: `/home/alucard/projects/Claude_X_Codex/.planning/token-log.jsonl` — 12 records; schema confirmed; 3 null `session_id` records confirmed and counted
- Live data: `/home/alucard/projects/The-Crucible/.planning/token-log.jsonl` — 4 records; cross-project aggregation confirmed
- Node.js v22.22.0 `fs.glob()`: tested live — returns `AsyncIterator`, `.then()` confirmed to throw
- Chart.js 4.5.1: `npm show chart.js version` confirmed; UMD file 204,390 bytes confirmed by download
- `fetch()` in Node.js v22: built-in global, no import required
- Linux `rename(2)`: POSIX standard; same-filesystem rename is atomic

### Secondary (MEDIUM confidence)

- Chart.js vs uPlot size comparison: https://www.luzmo.com/blog/plotly-js (2025) — bundle sizes and chart type support confirmed
- Dashboard KPI threshold: https://improvado.io/blog/dashboard-design-guide (2026) — 12-KPI engagement limit
- Dashboard interactivity tradeoff: https://www.smashingmagazine.com/2025/09/ux-strategies-real-time-dashboards/ — 35% time-to-insight increase with excessive interactivity
- ccusage feature set (for comparison): https://github.com/ryoppippi/ccusage (2026-04-02)

### Tertiary (carry-forward from v1.0, cited not re-verified)

- GitHub Issue #14303 — Codex CLI silent background hang (open as of CLI 0.114.0)
- CVE-2025-59536 — API key exfiltration via Claude Code project hooks (patched 2.0.65+)
- GitHub Issue #13186 — Codex Plus quota metering anomaly (March 2026)

---

*Research completed: 2026-04-02*
*Ready for roadmap: yes*
