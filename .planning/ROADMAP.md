# Roadmap: Claude X Codex

## Milestones

- ✅ **v1.0 Claude X Codex** — Phases 1-4 (shipped 2026-04-02) — [archive](milestones/v1.0-ROADMAP.md)
- ✅ **v1.1 Global Metrics Dashboard** — Phases 5-7 (shipped 2026-04-03) — [archive](milestones/v1.1-ROADMAP.md)
- ✅ **v2.0 Three-Model Intelligence** — Phases 8-14 (shipped 2026-04-03)
- 🔄 **v3.0 Adaptive Intelligence** — Phases 15-18 (in progress)

## Phases

<details>
<summary>✅ v1.0 Claude X Codex (Phases 1-4) — SHIPPED 2026-04-02</summary>

- [x] Phase 1: Foundation (3/3 plans) — completed 2026-04-02
- [x] Phase 2: Review Gate & GSD Integration (2/2 plans) — completed 2026-04-02
- [x] Phase 3: Plan Review Loop & Superpowers (2/2 plans) — completed 2026-04-02
- [x] Phase 4: Cost Reporting (1/1 plan) — completed 2026-04-02

</details>

<details>
<summary>✅ v1.1 Global Metrics Dashboard (Phases 5-7) — SHIPPED 2026-04-03</summary>

- [x] Phase 5: Data Pipeline (3/3 plans) — completed 2026-04-03
- [x] Phase 6: Dashboard Generator (2/2 plans) — completed 2026-04-03
- [x] Phase 7: Charts & Hook Integration (2/2 plans) — completed 2026-04-03

</details>

<details>
<summary>✅ v2.0 Three-Model Intelligence (Phases 8-14) — SHIPPED 2026-04-03</summary>

- [x] Phase 8: MiniMax Foundation (3/3 plans) — completed 2026-04-03
- [x] Phase 9: Dual Review Gate (1/1 plan) — completed 2026-04-03
- [x] Phase 10: Adversarial Plan Review (2/2 plans) — completed 2026-04-03
- [x] Phase 11: PostToolUse Bug Scanner (1/1 plan) — completed 2026-04-03
- [x] Phase 12: Context Compression (2/2 plans) — completed 2026-04-03
- [x] Phase 13: Codex Execution Pipeline (1/1 plan) — completed 2026-04-03
- [x] Phase 14: Three-Model Reporting (2/2 plans) — completed 2026-04-03

</details>

### v3.0 Adaptive Intelligence (Phases 15-18)

- [x] **Phase 15: Decision Capture Infrastructure** — Structured logging of every routing decision; task-type taxonomy; dismiss command; freeze flag (completed 2026-04-04)
- [ ] **Phase 16: Analysis Engine** — Read-only statistical analysis; noise profiles; per-project weights; SessionStart analyzer
- [ ] **Phase 17: Auto-Tuning + Confidence Gate** — Atomic config writer; safety bounds; confidence gate; routing audit log
- [ ] **Phase 18: Cross-Project Intelligence + Observability** — Global hypothesis engine; experiment design; adaptive dashboard panels

**Phase dependencies:**
- Phase 15 (data capture) must complete first — all downstream phases depend on structured decision logs
- Phase 16 depends on Phase 15 (needs data to analyze; read-only validation before Phase 17 can write)
- Phase 17 depends on Phase 16 (analyzer must produce validated recommendations before config writer is wired)
- Phase 18 depends on Phase 16 (hypotheses require cross-project data; dashboard panels require Phase 15-16 signals)

## Phase Details

### Phase 8: MiniMax Foundation

Add MiniMax M-2.7 as a model provider in the hook infrastructure. Create `minimax-exec.js` shared module (OpenAI SDK wrapper with `baseURL: https://api.minimax.io/v1`). Add MiniMax pricing to `codex-pricing.js` ($0.30/$1.20 input/output, $0.06 cache read). Fix Opus 4.6 pricing (currently $15/$75 which is Opus 4.1 — should be $5/$25). Set up `MINIMAX_API_KEY` environment variable. Update project settings with MiniMax config block. Verify connectivity with a test call.

### Phase 9: Dual Review Gate

Modify `codex-review-gate.js` (Stop hook) to run Codex and MiniMax reviews in parallel via `Promise.all`. Both produce independent verdicts. Merge: BLOCK if either flags an issue. Token logging tracks both models separately. Advisory output shows which model(s) flagged what. If Codex is rate-limited, MiniMax review still runs independently (graceful degradation).

### Phase 10: Adversarial Plan Review

Modify `codex-multi-round-reviewer.js` to accept model-per-round configuration. Round 1 stays Codex (constructive review). Round 2 switches to MiniMax (adversarial/devil's advocate — poke holes, find flaws, challenge assumptions). MiniMax's different reasoning patterns (self-evolution trained) make it a genuinely different adversary vs Codex doing both rounds. Must handle `<think>` tag preservation when passing Round 1 findings to MiniMax in Round 2. Applies to both GSD and Superpowers plan reviewers.

### Phase 11: PostToolUse Bug Scanner

Create `minimax-post-scan.js` — lightweight MiniMax bug/security scan after every Write/Edit via PostToolUse hook. Advisory only (additionalContext), never blocking. Cost: $0.01-0.03 per scan. Max_tokens capped at 1000 to control verbosity. Integrated with existing `codex-token-logger.js` for tracking.

### Phase 12: Context Compression

Create `minimax-compress.js` — a general-purpose MiniMax compression utility for anything with large token counts. Use cases: large git diffs before review, long conversation context when approaching limits (integrate with gsd-context-monitor), big API/tool output before injection as additionalContext, large file reads before Opus reasoning, plan review input compression. Hook integration via `UserPromptSubmit` and `PostToolUse`. Also usable as a require() utility from any hook script.

### Phase 13: Codex Execution Pipeline

Modify gsd-executor agent from a code-writing Opus subagent into a thin orchestrator. For each plan task: generate a handoff spec → invoke Codex CLI (`codex exec --full-auto --json`) → validate output → commit atomically. Fallback chain: Codex CLI (free via subscription) → MiniMax API ($0.30/$1.20, orchestrator writes files since MiniMax has no filesystem access) → prompt user (fail-closed). Detects Codex rate limits via exit codes, stderr messages, HTTP 429 in JSONL, and rate_limit_pct >= 95. The orchestrator can run on Sonnet (cheaper than Opus) since it only generates handoff specs, not code.

### Phase 14: Three-Model Reporting

Update `codex-token-logger.js` to recognize MiniMax model entries. Update `codex-cost-reporter.js` for three-model savings reports (Codex + MiniMax vs Opus-only baseline). Update `codex-global-aggregator.js` to aggregate MiniMax data from token logs. Update `codex-dashboard-generator.js` with three-model charts, per-model breakdowns, and fallback event tracking.

### Phase 15: Decision Capture Infrastructure

**Goal**: The system records every routing and review decision in a structured, queryable format that unblocks all downstream analysis.
**Depends on**: Phase 14 (extends existing token-log.jsonl schema)
**Requirements**: DCAP-01, DCAP-02, DCAP-03, DCAP-04, DCAP-05
**Success Criteria** (what must be TRUE):
  1. After any model call, token-log.jsonl contains outcome (accepted/dismissed/committed), latency_ms, and task-type fields alongside existing data — verifiable by reading the file
  2. User can run `/gsd:dismiss-last` after a false-positive review block and a negative training signal record appears in decision-log.jsonl
  3. Task types are categorized into 12 distinct categories (not 4) — verifiable by checking the task_type field across a session's log entries
  4. Setting `"adaptive": false` in settings.json causes the system to skip all adaptive behavior and operate on static rules — verifiable by adding the flag and running a session
**Plans**: 2 plans

Plans:
- [x] 15-01-PLAN.md — Core decision capture infrastructure (hook-signal.js, decision-logger.js, upstream hook mods, 12-category taxonomy)
- [x] 15-02-PLAN.md — User commands (/gsd:dismiss-last, /gsd:freeze, /gsd:unfreeze)

### Phase 16: Analysis Engine

**Goal**: The system reads accumulated decision data and produces statistical recommendations about routing and review thresholds, without touching any config file.
**Depends on**: Phase 15 (requires structured decision data to analyze)
**Requirements**: ANLZ-01, ANLZ-02, ANLZ-03, ANLZ-04
**Success Criteria** (what must be TRUE):
  1. After a SessionStart, `recommendations.json` exists and contains weighted statistics per tunable parameter — verifiable by reading the file
  2. A review rule that has been dismissed 3 times in 30 days for a project is suppressed in that project's noise profile — verifiable by triggering the condition and checking that subsequent sessions skip the rule
  3. When a git commit follows a session without edits to Claude's output, the matching decision-log.jsonl record has `committed: true` — verifiable by checking the log after a clean commit
  4. Per-project routing weights (keyed by project path prefix) are read by the analyzer and reflected in its recommendations — verifiable by setting a project weight and observing a recommendation change
**Plans**: 2 plans

Plans:
- [x] 15-01-PLAN.md — Core decision capture infrastructure (hook-signal.js, decision-logger.js, upstream hook mods, 12-category taxonomy)
- [ ] 15-02-PLAN.md — User commands (/gsd:dismiss-last, /gsd:freeze, /gsd:unfreeze)

### Phase 17: Auto-Tuning + Confidence Gate

**Goal**: The system applies validated recommendations to live config files atomically, with safety bounds enforced, and only when statistical confidence is sufficient.
**Depends on**: Phase 16 (config writer is called only by the analyzer; analyzer must be validated read-only first)
**Requirements**: TUNE-01, TUNE-02, TUNE-03, TUNE-04
**Success Criteria** (what must be TRUE):
  1. A config change applied by the system appears atomically — no partial write is ever visible during the rename — and the previous value is preserved in adjustment-log.jsonl with before/after values and confidence score
  2. A parameter value the analyzer recommends setting outside its hard bounds (e.g., scan threshold below 1 or above 100) is clamped at the boundary, never written as an out-of-range value
  3. With fewer than 30 events per parameter or confidence below 0.8, the system logs the recommendation as advisory only and makes no config change — verifiable by inspecting adjustment-log.jsonl
  4. The routing audit log records the reason each call was routed to a specific model — verifiable by opening the log and reading why a recent decision was made
**Plans**: 2 plans

Plans:
- [ ] 15-01-PLAN.md — Core decision capture infrastructure (hook-signal.js, decision-logger.js, upstream hook mods, 12-category taxonomy)
- [ ] 15-02-PLAN.md — User commands (/gsd:dismiss-last, /gsd:freeze, /gsd:unfreeze)
**UI hint**: yes

### Phase 18: Cross-Project Intelligence + Observability

**Goal**: The system learns from patterns across all projects on this machine and surfaces those insights — including active hypotheses and experiment proposals — in the dashboard.
**Depends on**: Phase 16 (needs cross-project decision data flowing; dashboard panels need Phase 15-16 signals)
**Requirements**: XPRJ-01, XPRJ-02, XPRJ-03, XPRJ-04, OBSV-01, OBSV-02
**Success Criteria** (what must be TRUE):
  1. The global aggregator collects decision logs from every project with the three-model router installed — verifiable by checking that `global-decision-log.jsonl` contains entries from multiple projects
  2. After enough cross-project data accumulates, the system generates at least one hypothesis (e.g., "MiniMax outperforms Codex on security reviews in this codebase") and writes it to a hypotheses file — verifiable by reading the file
  3. A proposed experiment to test a hypothesis is presented to the user for approval before any code or config is changed — verifiable by running the experiment proposal flow and confirming no changes occur without approval
  4. The dashboard shows a panel with dismiss rate, false-positive trend, and routing efficiency over time — verifiable by opening dashboard.html and seeing the panel with real data
  5. The dashboard shows active hypotheses, experiment status, and cross-project insights — verifiable by opening dashboard.html and seeing the panel populated
**Plans**: 2 plans

Plans:
- [ ] 15-01-PLAN.md — Core decision capture infrastructure (hook-signal.js, decision-logger.js, upstream hook mods, 12-category taxonomy)
- [ ] 15-02-PLAN.md — User commands (/gsd:dismiss-last, /gsd:freeze, /gsd:unfreeze)
**UI hint**: yes

## Progress

| Phase | Milestone | Plans Complete | Status | Completed |
|-------|-----------|----------------|--------|-----------|
| 1. Foundation | v1.0 | 3/3 | Complete | 2026-04-02 |
| 2. Review Gate & GSD Integration | v1.0 | 2/2 | Complete | 2026-04-02 |
| 3. Plan Review Loop & Superpowers | v1.0 | 2/2 | Complete | 2026-04-02 |
| 4. Cost Reporting | v1.0 | 1/1 | Complete | 2026-04-02 |
| 5. Data Pipeline | v1.1 | 3/3 | Complete | 2026-04-03 |
| 6. Dashboard Generator | v1.1 | 2/2 | Complete | 2026-04-03 |
| 7. Charts & Hook Integration | v1.1 | 2/2 | Complete | 2026-04-03 |
| 8. MiniMax Foundation | v2.0 | 3/3 | Complete | 2026-04-03 |
| 9. Dual Review Gate | v2.0 | 1/1 | Complete | 2026-04-03 |
| 10. Adversarial Plan Review | v2.0 | 2/2 | Complete | 2026-04-03 |
| 11. PostToolUse Bug Scanner | v2.0 | 1/1 | Complete | 2026-04-03 |
| 12. Context Compression | v2.0 | 2/2 | Complete | 2026-04-03 |
| 13. Codex Execution Pipeline | v2.0 | 1/1 | Complete | 2026-04-03 |
| 14. Three-Model Reporting | v2.0 | 2/2 | Complete | 2026-04-03 |
| 15. Decision Capture Infrastructure | v3.0 | 2/2 | Complete   | 2026-04-04 |
| 16. Analysis Engine | v3.0 | 0/? | Not started | - |
| 17. Auto-Tuning + Confidence Gate | v3.0 | 0/? | Not started | - |
| 18. Cross-Project Intelligence + Observability | v3.0 | 0/? | Not started | - |
