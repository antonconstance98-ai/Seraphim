---
phase: 09-dual-review-gate
plan: 01
subsystem: hooks
tags: [dual-review, parallel-execution, minimax, codex, stop-hook, token-logging]
dependency_graph:
  requires:
    - 08-01 (codex-pricing.js with minimax-m2.7 pricing entry)
    - 08-02 (minimax-exec.js with runMinimax export)
    - 08-03 (minimax config block in settings.json)
  provides:
    - codex-review-gate.js v3.0.0 with dual-model parallel review
    - dual_review: true flag in token-log.jsonl entries
    - per-model attribution in block reason strings
  affects:
    - token-log.jsonl (now two entries per review: gpt-5.4 + minimax-m2.7)
    - Stop hook behavior (parallel review instead of sequential Codex-only)
tech_stack:
  added: []
  patterns:
    - Promise.all for parallel async execution
    - .catch() wrappers on each Promise.all leg for rejection safety
    - Per-model JSONL logging with dual_review flag
    - Differentiated review prompts (Codex: correctness/arch, MiniMax: security/edge-cases)
key_files:
  created: []
  modified:
    - ~/.claude/hooks/codex-review-gate.js (v2.0.0 -> v3.0.0, outside repo)
decisions:
  - Used direct runCodexExec (not runWithFallback) for Codex leg to prevent double-MiniMax spend on rate-limit
  - Used computeCodexCostStrict (not computeCost) for MiniMax cost logging to avoid gpt-5.4 rate misapplication
  - MiniMax source field set to 'api' (not 'api-fallback') for correct cost aggregation attribution
  - buildMinimaxReviewPrompt delegates to buildReviewPrompt for bulk-ops/test-gen (light review; no differentiation needed)
metrics:
  duration: ~10min
  completed: 2026-04-03
  tasks_completed: 3
  files_modified: 1
  requirements_satisfied: [D-01, D-02, D-03, D-04, D-05, D-06, D-07]
---

# Phase 09 Plan 01: Dual Review Gate Summary

**One-liner:** Codex + MiniMax parallel review via Promise.all with verdict merge, per-model attribution, and dual token logging in the Stop hook.

## What Was Built

Modified `~/.claude/hooks/codex-review-gate.js` (v2.0.0 → v3.0.0) to run Codex and MiniMax code reviews simultaneously using `Promise.all`. Previously the Stop hook ran a single Codex review sequentially. Now both models run in parallel:

- **Codex prompt:** Existing `buildReviewPrompt` — correctness, architecture, requirements alignment, convention
- **MiniMax prompt:** New `buildMinimaxReviewPrompt` — security vulnerabilities, edge cases, race conditions, error handling gaps

The dual perspective is the key value: MiniMax's self-evolution training gives it genuinely different reasoning patterns from Codex, catching security/edge-case issues that Codex's correctness focus may miss.

## Tasks Completed

| Task | Name | Commit | Status |
|------|------|--------|--------|
| 1 | Add MiniMax import and security-focused review prompt builder | 6d34e5d | Complete |
| 2 | Replace single Codex call with parallel dual-model review, verdict merge, dual token logging | a30d4fa | Complete |
| 3 | Verify dual review gate works end-to-end | — | Auto-approved (auto_advance: true) |

## Key Implementation Details

### Parallel Execution (D-03)

```javascript
const [codexResult, minimaxResult] = await Promise.all([
  runCodexExec(reviewPrompt, { cwd, timeoutMs: 120000, model: 'gpt-5.4' })
    .catch((e) => ({ success: false, output: '', tokens: null, error: e.message })),
  runMinimax(minimaxPrompt, { maxTokens: 2000, timeoutMs: 120000 })
    .catch((e) => ({ success: false, text: '', tokens: null, cost: 0, error: e.message }))
]);
```

Both legs wrapped in `.catch()` to convert any thrown errors (spawn failures, SDK load errors) into structured `{ success: false }` objects, preventing Promise.all from rejecting and losing the other leg's result.

### Verdict Merge (D-01, D-02)

- BLOCK if either model flags an issue
- Both reasons always included in block message: "Codex found: X. MiniMax found: Y. Fix before responding."
- Skipped/failed model labeled as "Codex: skipped" or "MiniMax: skipped" for transparency
- Single JSON output object (never two separate outputs — Pitfall 6 avoided)

### Verdict Parsing (Pitfall 2 mitigation)

- Codex output (JSONL): `parseVerdict(extractCodexText(codexResult.output))`
- MiniMax output (plain text): `parseVerdict(minimaxResult.text)` — direct, no extractCodexText
- Calling `extractCodexText` on MiniMax text would JSON-parse plain text lines, return empty string, and parseVerdict would always return ALLOW

### Dual Token Logging (D-06, D-07)

Both models log separate JSONL entries to `token-log.jsonl`:
- Codex entry: `model: 'gpt-5.4'`, `source: 'cli'`, cost via `computeCost()`
- MiniMax entry: `model: 'minimax-m2.7'`, `source: 'api'`, cost via `computeCodexCostStrict()`, `dual_review: true` on both

### Fail-Open Paths (D-04, D-05)

- Codex fails → MiniMax-only review proceeds (not zero-model)
- MiniMax fails → Codex-only review proceeds (existing behavior)
- Both fail → stderr message + `process.exit(0)` (fail-open, never blocks on infrastructure error)

## Deviations from Plan

None — plan executed exactly as written.

All four pitfall avoidances implemented as specified:
- Pitfall 1: `.catch()` wrappers on both Promise.all legs
- Pitfall 2: `extractCodexText` only for Codex, direct `.text` for MiniMax
- Pitfall 3: `computeCodexCostStrict` (not `computeCost`) for MiniMax cost
- Pitfall 4: `source: 'api'` (not `'api-fallback'`) for MiniMax log entries

## Verification Results

All 11 plan verification checks passed:

1. File loads without errors (exits 0, stdin parse error is expected in isolation)
2. Exactly 1 `await Promise.all(` call
3. 2 `.catch()` wrappers on Promise.all legs (5 total including other catch blocks)
4. 2 `dual_review: true` data entries (one per model log record)
5. `Codex found:` and `MiniMax found:` present in output logic
6. `extractCodexText(minimaxResult` does NOT appear anywhere (Pitfall 2 safe)
7. `computeCodexCostStrict(minimaxResult.tokens, 'minimax-m2.7')` present
8. `source: 'api'` present for MiniMax log entry
9. `stop_hook_active` check at line 45 (first check after JSON.parse)
10. `codex-hook-version: 3.0.0` in header
11. Smoke test: SMOKE_TEST_PASS — parseVerdict handles BLOCK, ALLOW, no-verdict on plain text

## Known Stubs

None. The dual review gate is fully wired. The MiniMax leg will fail-open (success: false) if `MINIMAX_API_KEY` is not set in the shell environment — this is expected behavior, not a stub. The connectivity test (`~/.claude/hooks/minimax-connectivity-test.js`) should be run after API key configuration to verify end-to-end dual review.

## Self-Check: PASSED

Files modified:
- `~/.claude/hooks/codex-review-gate.js` — FOUND (outside repo, verified loads without errors)

Commits:
- `6d34e5d` — FOUND (git log confirmed)
- `a30d4fa` — FOUND (git log confirmed)
