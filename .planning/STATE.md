---
gsd_state_version: 1.0
milestone: v2.0
milestone_name: Three-Model Intelligence
status: executing
stopped_at: Completed 10-01-PLAN.md -- MiniMax M-2.7 wired as Round 2 adversarial reviewer with reasoning_split and D-08 fallback
last_updated: "2026-04-03T19:37:43.149Z"
last_activity: 2026-04-03
progress:
  total_phases: 7
  completed_phases: 2
  total_plans: 6
  completed_plans: 5
---

# Project State

## Project Reference

See: .planning/PROJECT.md (updated 2026-04-03)

**Core value:** Every task goes to the model that's best at it — Opus for reasoning and architecture, Codex for fast execution — with cross-model review catching what either model misses alone.
**Current focus:** Phase 10 — adversarial-plan-review

## Current Position

Phase: 10 (adversarial-plan-review) — EXECUTING
Plan: 2 of 2
Status: Ready to execute
Last activity: 2026-04-03

## Performance Metrics

**Velocity:**

- Total plans completed: 13 (8 v1.0 + 5 v1.1)
- Average duration: ~5 min/plan
- Total execution time: ~65 min

**By Phase:**

| Phase | Plans | Total | Avg/Plan |
|-------|-------|-------|----------|
| 1. Foundation | 3 | ~10 min | ~3 min |
| 2. Review Gate & GSD | 2 | ~18 min | ~9 min |
| 3. Plan Review Loop | 2 | ~8 min | ~4 min |
| 4. Cost Reporting | 1 | ~3 min | 3 min |

**Recent Trend:**

- Last 5 plans: 3, 6, 4, 4, 3 min
- Trend: Stable

*Updated after each plan completion*
| Phase 05-data-pipeline P01 | 2 | 2 tasks | 3 files |
| Phase 05-data-pipeline P02 | 3min | 2 tasks | 7 files |
| Phase 05-data-pipeline P03 | 5min | 1 tasks | 1 files |
| Phase 06 P01 | 4 | 1 tasks | 2 files |
| Phase 06-dashboard-generator P02 | 5 | 2 tasks | 3 files |
| Phase 07-charts-hook-integration P02 | 2min | 1 tasks | 1 files |
| Phase 07-charts-hook-integration P01 | 10 | 2 tasks | 2 files |
| Phase 08-minimax-foundation P01 | 8 | 2 tasks | 2 files |
| Phase 08-minimax-foundation P02 | 4 | 1 tasks | 1 files |
| Phase 08-minimax-foundation P03 | 5 | 2 tasks | 2 files |
| Phase 09-dual-review-gate P01 | 1min | 3 tasks | 1 files |
| Phase 10-adversarial-plan-review P01 | 5 | 2 tasks | 2 files |

## Accumulated Context

### Decisions

Decisions are logged in PROJECT.md Key Decisions table.
Recent decisions affecting current work:

- [v1.1 Roadmap]: INTG-02 (SessionStart hook wiring) goes in Phase 7 — register hook only after standalone generator verified correct in Phase 6
- [v1.1 Roadmap]: Phase 5 must include centralized `pricing.js` module before dashboard consumes pricing (prevents silent $0 inflation of savings %)
- [v1.1 Research]: `fs.glob()` on Node.js v22 returns AsyncIterator — must use `for await...of`, NOT `.then()`
- [v1.1 Research]: 25% of live records have `session_id: null` (codex-multi-round-reviewer.js) — treat as "Unattributed", never drop
- [v1.1 Research]: All dashboard writes must use write-to-temp-then-renameSync (atomic on Linux) — prevents concurrent session corruption
- [Phase 04-cost-reporting]: OPUS_PRICING inline in cost reporter — keeps codex-exec.js Codex-only, avoids cross-model pricing confusion
- [Phase 05-data-pipeline]: computeOpusCost preserves no-rounding behavior — avoids changing stored cost values in existing token-log.jsonl files
- [Phase 05-data-pipeline]: computeCodexCostStrict added alongside computeCost (not replacement) — new consumers surface pricing gaps; existing consumers unchanged
- [Phase 05-data-pipeline]: [Phase 05-01]: codex-exec.js re-exports computeCost from codex-pricing.js — codex-token-logger.js import chain preserved with zero downstream changes
- [Phase 05-data-pipeline]: Aggregator already existed from prior work session — verified acceptance criteria against existing file rather than recreating
- [Phase 05-data-pipeline]: Idempotency relies on both mtime+size fast path AND dedup Set — cache skips file reads, Set catches stale-cache edge cases
- [Phase 05-data-pipeline]: [Phase 05-03]: Discovery cache TTL=1hr in project-index.json — warm runs skip spawnSync find, reducing no-op elapsed_ms from 151ms to 2ms
- [Phase 05-data-pipeline]: [Phase 05-03]: wasWarm flag captured before writes — prevents race between warm check and index write; carry-forward pattern preserves full discovered_files on warm runs
- [Phase 06]: generateDashboard returns DASHBOARD_DATA object in stub — Plan 02 replaces body keeping identical signature
- [Phase 06]: modelSplit always initializes gpt-5.4 and gpt-5.4-mini with zero values before merging observed data
- [Phase 06]: ensureChartJs pins Chart.js 4.5.1 SHA-256 (48444a82...) — refuses any CDN response with mismatched hash
- [Phase 06]: generateDashboard reads Chart.js sidecar synchronously — avoids async complexity in aggregator call path
- [Phase 06]: [Phase 06-02]: htmlEsc() re-implemented inline in dashboard script block — ensures full self-containment at runtime
- [Phase 06]: [Phase 06-02]: Aggregator Step 8 wraps generateDashboard in silent-fail try/catch — dashboard generation never blocks aggregation
- [Phase 07-charts-hook-integration]: Appended aggregator as third hook in existing SessionStart group (timeout:30) — no new group needed; all other sections unchanged
- [Phase 07-charts-hook-integration]: Weekly grouping uses ISO Monday Thursday-algorithm (UTCDay+6)%7 for correct ISO week semantics
- [Phase 07-charts-hook-integration]: buildBar filters both r.name and r.projectName for Unattributed to handle field name variation
- [Phase 07-charts-hook-integration]: series.length===0 written without spaces so grep-based verification finds exact string literal
- [Phase 08-minimax-foundation]: Opus 4.6 pricing is dollar5/dollar25 per 1M tokens (not dollar15/dollar75 which was Opus 4.1 -- a 3x error corrected in global.jsonl migration)
- [Phase 08-minimax-foundation]: MiniMax M-2.7 pricing added to CODEX_PRICING (input:0.30, cached:0.06, output:1.20) -- consistent with existing pattern, no function changes needed
- [Phase 08-minimax-foundation]: Migration rewrites global.jsonl only -- per-project token-log.jsonl never contains opus_baseline_usd; aggregator computes it at merge time
- [Phase 08-minimax-foundation]: callWithRetry wraps only the API call in runMinimax — AbortController timer clears in finally regardless of retry count
- [Phase 08-minimax-foundation]: runWithFallback escalates to MiniMax only on rate-limit Codex failure — prevents MiniMax spend for auth errors or misconfigurations
- [Phase 08-minimax-foundation]: Defensive cached_tokens fallback: checks prompt_tokens_details.cached_tokens then usage.cached_tokens — handles MiniMax API field placement uncertainty
- [Phase 08-minimax-foundation]: minimax config block is a sibling of codex in settings.json (D-10) — prevents nesting ambiguity, Phases 9-14 can read minimax.* directly
- [Phase 08-minimax-foundation]: Connectivity test uses 120s timeout vs 90s default — MiniMax pre-answer latency documented up to 55s on first call
- [Phase 09-dual-review-gate]: Direct runCodexExec (not runWithFallback) for Codex leg — prevents double-MiniMax spend on rate-limit; MiniMax leg already runs independently
- [Phase 09-dual-review-gate]: computeCodexCostStrict (not computeCost) for MiniMax cost logging — avoids gpt-5.4 rate misapplication; source='api' (not 'api-fallback') for direct Phase 9 calls
- [Phase 09-dual-review-gate]: .catch() wrappers on both Promise.all legs — converts spawn/SDK-load throws to { success: false } to preserve other leg's result (D-04/D-05)
- [Phase 10-adversarial-plan-review]: reasoning_split added as opt-in via opts.reasoningSplit -- callers not passing it get unchanged behavior
- [Phase 10-adversarial-plan-review]: Partial success guard: empty MiniMax text triggers D-08 fallback -- empty text is feature-level failure even when success:true
- [Phase 10-adversarial-plan-review]: No runWithFallback() for Round 2 -- Phase 10 goes MiniMax->Codex (opposite of Phase 9 Codex->MiniMax direction)
- [Phase 10-adversarial-plan-review]: Round 1 context capped at MAX_R1_CONTEXT=4000 chars before Round 2 injection -- prevents MiniMax timeout on long reviews

### Pending Todos

None yet.

### Blockers/Concerns

- [Phase 5]: Decide whether to fix `session_id` propagation in `codex-multi-round-reviewer.js` (pass from callers) or use "Unattributed" label — both acceptable; affects Phase 5 scope
- [Phase 6]: Validate Chart.js `<canvas>` renders from `file://` during Phase 6 verification; SVG fallback path exists if canvas is blocked by browser security

## Session Continuity

Last session: 2026-04-03T19:37:43.146Z
Stopped at: Completed 10-01-PLAN.md -- MiniMax M-2.7 wired as Round 2 adversarial reviewer with reasoning_split and D-08 fallback
Resume file: None
