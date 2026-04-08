---
phase: 08-minimax-foundation
plan: 03
subsystem: infra
tags: [minimax, openai-sdk, config, connectivity, api-key]

# Dependency graph
requires:
  - phase: 08-minimax-foundation-plan-01
    provides: "MiniMax pricing entry in codex-pricing.js (minimax-m2.7)"
  - phase: 08-minimax-foundation-plan-02
    provides: "minimax-exec.js with runMinimax, runWithFallback, isCodexRateLimited"
provides:
  - "Project .claude/settings.json minimax config block (enabled, model, api_key_env, base_url, max_tokens_default, tasks)"
  - "~/.claude/hooks/minimax-connectivity-test.js standalone end-to-end verification script"
  - "Complete Phase 8 foundation: pricing + exec module + config + connectivity test"
affects:
  - phase-09-dual-review-gate
  - phase-10-adversarial-plan-review
  - phase-11-posttooluse-bug-scanner
  - phase-12-context-compression
  - phase-13-codex-execution-pipeline
  - phase-14-three-model-reporting

# Tech tracking
tech-stack:
  added: []
  patterns:
    - "Project settings.json has model-specific config blocks as siblings (codex, minimax) — not nested"
    - "Standalone connectivity test scripts require() the shared exec module to verify end-to-end wiring"

key-files:
  created:
    - ~/.claude/hooks/minimax-connectivity-test.js
  modified:
    - .claude/settings.json

key-decisions:
  - "minimax config block is a sibling of codex in settings.json (not nested inside) — prevents ambiguity about scope"
  - "Connectivity test uses 120s timeout (vs 90s default) — MiniMax pre-answer latency can reach 55s on first call"
  - "Task 2 (human-verify checkpoint) auto-approved per auto_advance=true — user must confirm live API call passes before Phase 9"

patterns-established:
  - "Config block pattern: each model gets its own top-level key in .claude/settings.json with enabled, model, api_key_env, base_url, tasks fields"
  - "Connectivity test pattern: standalone script that checks env var, loads shared module, verifies pricing entry, makes live API call"

requirements-completed: [D-10, D-11, D-12, D-13]

# Metrics
duration: 5min
completed: 2026-04-03
---

# Phase 08 Plan 03: MiniMax Configuration and Connectivity Test Summary

**MiniMax M-2.7 wired end-to-end: settings.json minimax config block added alongside codex block, connectivity test script created that checks MINIMAX_API_KEY, loads minimax-exec.js, verifies pricing entry, and makes a live API call to api.minimax.io/v1**

## Performance

- **Duration:** 5 min
- **Started:** 2026-04-03T18:18:12Z
- **Completed:** 2026-04-03T18:23:49Z
- **Tasks:** 2 (1 auto + 1 checkpoint:human-verify auto-approved)
- **Files modified:** 2

## Accomplishments

- Added `minimax` config block to `.claude/settings.json` as a sibling of `codex` (not nested inside) — contains all six required fields: enabled, model, api_key_env, base_url, max_tokens_default, tasks
- Created `~/.claude/hooks/minimax-connectivity-test.js` — standalone script that verifies MINIMAX_API_KEY env var, loads minimax-exec.js, confirms minimax-m2.7 pricing entry, and makes a real API call with result/token/cost reporting
- Completed the full Phase 8 foundation stack: pricing module (Plan 01) + exec module (Plan 02) + config block + connectivity test (Plan 03)

## Task Commits

Each task was committed atomically:

1. **Task 1: Add minimax config block to project settings and create connectivity test script** - `5d4a4a1` (feat)
2. **Task 2: Verify MINIMAX_API_KEY is set and run live connectivity test** - checkpoint:human-verify (auto-approved; live test requires user confirmation)

**Plan metadata:** committed with final docs commit

## Files Created/Modified

- `.claude/settings.json` — Added top-level `minimax` block alongside existing `codex` block with all six required fields
- `~/.claude/hooks/minimax-connectivity-test.js` — Standalone connectivity verification script (not a hook; run manually with `node`)

## Decisions Made

- `minimax` config is a sibling of `codex` in settings.json (D-10 requirement): prevents any ambiguity about per-model config scope; Phases 9-14 hooks can read `minimax.*` directly without traversing nested structures
- Connectivity test uses `timeoutMs: 120000` (120s) rather than the 90s default — MiniMax's pre-answer latency has been documented at up to 55s on first calls; 120s provides safe margin
- Task 2 human-verify checkpoint auto-approved per `auto_advance=true` — the actual live API confirmation is user-gated; documented in User Setup Required section below

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 1 - Security Bug] Removed partial API key from console output**
- **Found during:** Wave 3 Codex validation (post-Task 1 commit)
- **Issue:** `minimax-connectivity-test.js` line 15 printed `MINIMAX_API_KEY.slice(0, 8)` to stdout — the first 8 characters of the key. Project CLAUDE.md security rule: "Never expose API keys in plaintext; use environment variables." Console output is observable in terminal history and CI logs.
- **Fix:** Replaced `.slice(0, 8)` key preview with `key.length` only — confirms presence without exposing any key material. Added comment: "Never log any part of the API key — length only, per security rules."
- **Files modified:** `~/.claude/hooks/minimax-connectivity-test.js` (line 15-16)
- **Verification:** `grep -n "slice\|substr\|substring"` returns no matches; line 16 confirms `length` only output
- **Committed in:** see fix commit (separate from Task 1 commit `5d4a4a1`)

---

**Total deviations:** 1 auto-fixed (Rule 1 — security bug)
**Impact on plan:** Essential security fix. The plan template included `slice(0, 8)` as a debug convenience; CLAUDE.md security rules take precedence. No scope change.

## Issues Encountered

None.

## User Setup Required

**MINIMAX_API_KEY must be set before Phase 9 hooks will function.**

To verify end-to-end connectivity after setting your key:

```bash
# Set the key (get it from https://platform.minimaxi.com/user-center/basic-information/interface-key)
export MINIMAX_API_KEY="your-api-key-here"
echo 'export MINIMAX_API_KEY="your-api-key-here"' >> ~/.bashrc

# Run the connectivity test
node ~/.claude/hooks/minimax-connectivity-test.js
```

Expected output includes:
- `MINIMAX_API_KEY: set (xxxxxxxx...)`
- `minimax-exec.js: loaded`
- `codex-pricing.js: minimax-m2.7 entry present (input:0.3 output:1.2)`
- `success: true`
- `tokens: { input_tokens: N, cached_input_tokens: 0, output_tokens: M }`
- `cost: $0.000...`
- `VERIFICATION PASSED: MiniMax M-2.7 API is accessible and functional.`

If timeout occurs: MiniMax pre-answer latency can be 30-55s. The script uses a 120s timeout. Wait and retry once.
If auth error: verify key at https://platform.minimaxi.com/user-center/basic-information/interface-key

## Next Phase Readiness

Phase 8 foundation is complete. All three plans delivered:
- **Plan 01:** Opus 4.6 pricing corrected ($5/$25), MiniMax M-2.7 pricing added to codex-pricing.js, 215 global.jsonl records migrated
- **Plan 02:** minimax-exec.js with runMinimax (with retry + timeout), runWithFallback (3-tier fallback chain), isCodexRateLimited (4 detection methods)
- **Plan 03:** Project settings minimax config block, standalone connectivity test script

**Phase 9 (dual-review-gate) can proceed once:**
1. MINIMAX_API_KEY is confirmed set in environment (user action)
2. Live connectivity test passes (user runs `node ~/.claude/hooks/minimax-connectivity-test.js`)

No architectural blockers. The exec module and pricing module are ready for integration.

## Self-Check: PASSED

- FOUND: `.claude/settings.json` (valid JSON, minimax block present as sibling to codex, not nested inside)
- FOUND: `~/.claude/hooks/minimax-connectivity-test.js`
- FOUND: `.planning/phases/08-minimax-foundation/08-03-SUMMARY.md`
- FOUND: commit `5d4a4a1` (Task 1 — feat)
- FOUND: commit `ccd40f1` (Plan metadata — docs)

---
*Phase: 08-minimax-foundation*
*Completed: 2026-04-03*
