# Roadmap: Claude X Codex

## Milestones

- ✅ **v1.0 Claude X Codex** — Phases 1-4 (shipped 2026-04-02) — [archive](milestones/v1.0-ROADMAP.md)
- ✅ **v1.1 Global Metrics Dashboard** — Phases 5-7 (shipped 2026-04-03) — [archive](milestones/v1.1-ROADMAP.md)
- 🔄 **v2.0 Three-Model Intelligence** — Phases 8-14 (in progress)

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

### v2.0 Three-Model Intelligence (Phases 8-14)

- [x] **Phase 8: MiniMax Foundation** — Pricing module, SDK wrapper, Opus pricing fix, env config
  - **Goal:** Add MiniMax M-2.7 as a model provider with working SDK wrapper, corrected pricing, and verified API connectivity
  - **Plans:** 3 plans
  - Plans:
    - [x] 08-01-PLAN.md — Fix Opus pricing, add MiniMax pricing entry, migrate historical data
    - [x] 08-02-PLAN.md — Create minimax-exec.js shared module (runMinimax, runWithFallback, isCodexRateLimited)
    - [x] 08-03-PLAN.md — Add minimax config to project settings, verify live API connectivity
- [x] **Phase 9: Dual Review Gate** — Codex + MiniMax reviews in parallel on Stop hook, merged verdicts (completed 2026-04-03)
  - **Goal:** Run Codex and MiniMax reviews in parallel on the Stop hook with merged verdicts and dual token logging
  - **Plans:** 1 plan
  - Plans:
    - [x] 09-01-PLAN.md — Parallel dual-model review gate with verdict merge, differentiated prompts, and dual token logging
- [x] **Phase 10: Adversarial Plan Review** — MiniMax replaces Codex as adversarial (Round 2) in plan review; devil's advocate role (completed 2026-04-03)
  - **Goal:** Route Round 2 of multi-round plan review to MiniMax as adversarial reviewer with full reasoning chain transparency
  - **Plans:** 2 plans
  - Plans:
    - [x] 10-01-PLAN.md — Add reasoning_split to minimax-exec.js, route Round 2 to MiniMax in codex-multi-round-reviewer.js
    - [x] 10-02-PLAN.md — Update REVIEWS.md headers in GSD and Superpowers plan reviewers for dual-model attribution
- [ ] **Phase 11: PostToolUse Bug Scanner** — MiniMax bug/security scan after every Write/Edit ($0.01-0.03/scan)
- [ ] **Phase 12: Context Compression** — Universal MiniMax compression for large diffs, files, tool outputs, conversations
- [ ] **Phase 13: Codex Execution Pipeline** — gsd-executor becomes thin orchestrator; Codex CLI writes code; MiniMax API fallback on rate limits; prompt user as last resort
- [ ] **Phase 14: Three-Model Reporting** — Token logging, cost reports, and dashboard updated for Opus + Codex + MiniMax

**Phase dependencies:**
- Phase 8 (foundation) must complete first — all other phases depend on `minimax-exec.js` and pricing
- Phases 9, 10, 11, 12 can run in parallel after Phase 8
- Phase 13 depends on Phase 8 (MiniMax fallback needs the SDK wrapper)
- Phase 14 depends on all prior phases (needs to track all three models)

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
| 9. Dual Review Gate | v2.0 | 1/1 | Complete   | 2026-04-03 |
| 10. Adversarial Plan Review | v2.0 | 2/2 | Complete    | 2026-04-03 |
| 11. PostToolUse Bug Scanner | v2.0 | — | Pending | — |
| 12. Context Compression | v2.0 | — | Pending | — |
| 13. Codex Execution Pipeline | v2.0 | — | Pending | — |
| 14. Three-Model Reporting | v2.0 | — | Pending | — |
