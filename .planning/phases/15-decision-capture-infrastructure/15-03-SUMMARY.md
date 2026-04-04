---
phase: 15-decision-capture-infrastructure
plan: "03"
subsystem: hooks
tags: [gap-closure, token-log, decision-log, latency, outcome-field]
dependency_graph:
  requires: ["15-01", "15-02"]
  provides: ["DCAP-01-complete", "DCAP-02-complete"]
  affects: ["token-log.jsonl schema", "decision-log.jsonl Stop ALLOW records"]
tech_stack:
  added: []
  patterns: ["outcome: null sentinel in token-log records", "Math.max latency aggregation across parallel model legs"]
key_files:
  modified:
    - ~/.claude/hooks/codex-review-gate.js
    - ~/.claude/hooks/minimax-post-scan.js
  verified_unchanged:
    - ~/.claude/hooks/decision-logger.js
decisions:
  - "outcome: null used as sentinel at write time — outcome (accepted/dismissed/committed) can only be determined later via user action, not at review call time"
  - "Math.max(codexModelLatencyMs || 0, minimaxModelLatencyMs || 0) || null — trailing || null converts 0 back to null when both models returned no latency (both failed or skipped)"
  - "review_model_latency_ms written as combined signal in review-gate — decision-logger reads this single key, no changes to decision-logger needed"
  - "Gap 3 (DCAP-05 adaptive:false enforcement) intentionally deferred to Phases 16-17 per plan design"
metrics:
  duration_min: 3
  completed_date: "2026-04-03"
  tasks_completed: 2
  tasks_total: 2
  files_modified: 2
  files_verified: 1
---

# Phase 15 Plan 03: Decision Capture Infrastructure Gap Closure Summary

**One-liner:** Added `outcome: null` sentinel to all direct-write token-log records and wired `review_model_latency_ms` combined signal in review-gate so decision-logger ALLOW events have non-null latency.

## Tasks Completed

| Task | Name | Commit | Files |
|------|------|--------|-------|
| 1 | Add outcome field to review-gate and post-scan token-log records, add review_model_latency_ms signal | a6c351d | `~/.claude/hooks/codex-review-gate.js`, `~/.claude/hooks/minimax-post-scan.js` |
| 2 | Verify decision-logger wiring is complete end-to-end | a6c351d | `~/.claude/hooks/decision-logger.js` (verified unchanged) |

## Changes Made

### codex-review-gate.js (3 edits)

1. Added `outcome: null` to Codex token-log record after `rate_limit_pct`
2. Added `outcome: null` to MiniMax token-log record after `cost_usd`
3. Added combined latency signal after existing 5 writeHookSignal calls:
   ```javascript
   const reviewModelLatencyMs = Math.max(codexModelLatencyMs || 0, minimaxModelLatencyMs || 0) || null;
   writeHookSignal(cwd, eventId, 'review_model_latency_ms', reviewModelLatencyMs);
   ```

### minimax-post-scan.js (1 edit)

1. Added `outcome: null` to token-log record after `cost_usd`

### decision-logger.js (no changes)

Already reads `state.review_model_latency_ms` at line 140. The key mismatch was on the writer side (review-gate) — adding the combined signal in review-gate resolved the mismatch without touching decision-logger.

## Verification Results

| Check | Expected | Actual | Status |
|-------|----------|--------|--------|
| `outcome: null` count in review-gate | 2 | 2 | PASS |
| `outcome: null` count in post-scan | 1 | 1 | PASS |
| `writeHookSignal.*review_model_latency_ms` in review-gate | 1 | 1 | PASS |
| `state.review_model_latency_ms` in decision-logger | 1 | 1 | PASS |
| review-gate syntax check | no SyntaxError | PASS | PASS |
| post-scan syntax check | no SyntaxError | PASS | PASS |
| Key string match (writer vs reader) | both `review_model_latency_ms` | WIRED | PASS |

## Gaps Closed

**DCAP-01 (outcome field in token-log):** All token-log records from direct-write hooks now include `outcome: null`. The value is null at write time by design — outcome (accepted/dismissed/committed) is determined via user action through `/gsd:dismiss-last`, not at review call time. This aligns with ROADMAP SC1 while being honest about the temporal constraint.

**DCAP-02 (review_model_latency_ms key mismatch):** Review-gate previously wrote `review_codex_latency_ms` and `review_minimax_latency_ms` to shared state but decision-logger read `review_model_latency_ms`. ALLOW event records always had `review_model_latency_ms: null`. Fixed by adding a combined signal computed as `Math.max(per-model latencies) || null` — decision-logger now reads a real value for ALLOW events when at least one model ran.

## Intentionally Deferred

**DCAP-05 (adaptive:false enforcement, Gap 3):** The freeze/unfreeze infrastructure is in place (Phase 15-02). The skip-behavior enforcement (hooks checking the flag and exiting) is a Phase 16-17 responsibility. Not touched in this plan.

## Deviations from Plan

None — plan executed exactly as written. decision-logger.js required no changes (confirmed per plan's contingency note).

## Known Stubs

None — both gap fixes are complete and wired end-to-end.

## Self-Check: PASSED

- Modified files exist: `~/.claude/hooks/codex-review-gate.js` — FOUND
- Modified files exist: `~/.claude/hooks/minimax-post-scan.js` — FOUND
- Commit a6c351d exists: confirmed via `git log --oneline`
- All verification criteria met (see table above)
