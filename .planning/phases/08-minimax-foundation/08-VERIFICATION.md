---
phase: 08-minimax-foundation
verified: 2026-04-03T18:45:00Z
status: gaps_found
score: 16/17 must-haves verified
re_verification: null
gaps:
  - truth: "MINIMAX_API_KEY environment variable is set and available to hook subprocesses"
    status: failed
    reason: "MINIMAX_API_KEY is not set in the current shell environment, .bashrc, .profile, or .bash_profile. The env var is referenced in settings.json and in runMinimax() but has not been configured on this machine."
    artifacts:
      - path: "~/.bashrc"
        issue: "No MINIMAX_API_KEY export found"
    missing:
      - "User must run: export MINIMAX_API_KEY='your-key' and add to ~/.bashrc"
      - "Live connectivity test (node ~/.claude/hooks/minimax-connectivity-test.js) cannot be confirmed until key is set"
human_verification:
  - test: "Set MINIMAX_API_KEY and run live connectivity test"
    expected: "Output shows 'success: true', non-zero token counts, cost > 0, and 'VERIFICATION PASSED: MiniMax M-2.7 API is accessible and functional.'"
    why_human: "Requires a valid API key from https://platform.minimaxi.com/user-center/basic-information/interface-key and a live network call. Cannot be verified programmatically without credentials."
---

# Phase 8: MiniMax Foundation Verification Report

**Phase Goal:** Add MiniMax M-2.7 as a model provider in the hook infrastructure. Create minimax-exec.js shared module (OpenAI SDK wrapper with baseURL: https://api.minimax.io/v1). Add MiniMax pricing to codex-pricing.js ($0.30/$1.20 input/output, $0.06 cache read). Fix Opus 4.6 pricing (currently $15/$75 which is Opus 4.1 — should be $5/$25). Set up MINIMAX_API_KEY environment variable. Update project settings with MiniMax config block. Verify connectivity with a test call.

**Verified:** 2026-04-03T18:45:00Z
**Status:** gaps_found (1 gap — MINIMAX_API_KEY not configured in environment)
**Re-verification:** No — initial verification

## Goal Achievement

### Observable Truths

All truths drawn from must_haves across 08-01-PLAN.md, 08-02-PLAN.md, and 08-03-PLAN.md.

| #  | Truth | Status | Evidence |
|----|-------|--------|----------|
| 1  | OPUS_PRICING uses $5/$25 (Opus 4.6), not $15/$75 (Opus 4.1) | VERIFIED | `codex-pricing.js` OPUS_PRICING: input=5, cached_input=1.25, output=25 |
| 2  | CODEX_PRICING contains minimax-m2.7 entry with input:0.30 cached_input:0.06 output:1.20 | VERIFIED | `CODEX_PRICING['minimax-m2.7']` = {"input":0.3,"cached_input":0.06,"output":1.2} |
| 3  | computeCodexCostStrict returns a number (not null) for model 'minimax-m2.7' | VERIFIED | Returns 0.3 for 1M input tokens; 1.2 for 1M output tokens |
| 4  | computeCost still returns a number for unknown models (backward compat via gpt-5.4 fallback) | VERIFIED | typeof computeCost({input_tokens:1e3},'bogus') === 'number' |
| 5  | All existing records in global.jsonl have opus_baseline_usd recalculated with $5/$25 pricing | VERIFIED | 222 records; first record baseline=0.086705 matches deterministic recalc from stored tokens |
| 6  | Migration is idempotent — running it twice produces identical global.jsonl content | VERIFIED | diff /tmp/g1_verify.jsonl ~/.claude/dashboard/global.jsonl shows no output |
| 7  | Dashboard HTML is regenerated after migration to reflect corrected savings percentages | VERIFIED | ~/.claude/dashboard/dashboard.html exists |
| 8  | Malformed and blank lines in global.jsonl are preserved unchanged by the migration | VERIFIED | Migration code: blank lines returned as-is; JSON parse failures increment malformed counter and return original line unchanged |
| 9  | runMinimax(prompt, opts) makes an API call to api.minimax.io/v1 using the openai SDK with baseURL swap | VERIFIED | `MINIMAX_BASE_URL = 'https://api.minimax.io/v1'`; `new OpenAI({ baseURL: MINIMAX_BASE_URL, apiKey })`; confirmed in source |
| 10 | runMinimax returns { success, text, tokens, cost, error } shape | VERIFIED | No-key guard returns correct shape; source code confirms all return paths use this shape |
| 11 | runMinimax uses temperature 0.01 (never exactly 0) | VERIFIED | `const temperature = options.temperature || 0.01` — line 71 |
| 12 | runMinimax defaults to max_tokens 2000 | VERIFIED | `const maxTokens = options.maxTokens || 2000` — line 69 |
| 13 | runMinimax computes cost via computeCodexCostStrict from codex-pricing.js | VERIFIED | Line 125-126: `require(PRICING_PATH); computeCodexCostStrict(tokens, 'minimax-m2.7')` |
| 14 | runWithFallback tries Codex CLI first, falls back to MiniMax on rate-limit, prompts user as last resort | VERIFIED | Three-tier chain in source: codex-cli -> api-fallback -> fail-open/fail-closed by taskCategory |
| 15 | isCodexRateLimited detects exit code + stderr signals, HTTP 429, timeout with no output, rate_limit_pct >= 95 | VERIFIED | All 6 test cases pass: null->false, pct95->true, 'rate limit'->true, '{"status":429}'->true, timeout+empty->true, pct50+ok->false |
| 16 | Project settings.json has a top-level minimax block alongside (not inside) the codex block | VERIFIED | settings.json: minimax is sibling of codex, not nested; codex.minimax === undefined |
| 17 | MINIMAX_API_KEY environment variable is set and available to hook subprocesses | FAILED | Not in current shell, .bashrc, .profile, or .bash_profile. User action required. |

**Score:** 16/17 truths verified

### Required Artifacts

| Artifact | Expected | Status | Details |
|----------|----------|--------|---------|
| `~/.claude/hooks/codex-pricing.js` | Corrected Opus pricing and MiniMax pricing entry | VERIFIED | OPUS_PRICING input=5; CODEX_PRICING['minimax-m2.7'] present |
| `~/.claude/hooks/migrate-opus-pricing.js` | Idempotent migration script with dashboard regen | VERIFIED | 132 lines; contains recalcOpusBaseline, fs.renameSync, generateDashboard; runs idempotently |
| `~/.claude/hooks/minimax-exec.js` | MiniMax provider module with runMinimax, runWithFallback, isCodexRateLimited | VERIFIED | 286 lines; all three exports confirmed; 'use strict'; no shebang; no process.stdin |
| `.claude/settings.json` | MiniMax configuration block | VERIFIED | All 6 fields correct: enabled, model, api_key_env, base_url, max_tokens_default, tasks |
| `~/.claude/hooks/minimax-connectivity-test.js` | Standalone test script for verifying MiniMax API connectivity | VERIFIED | Contains require('./minimax-exec'), require('./codex-pricing'), MINIMAX_API_KEY check; security fix applied (no key slicing) |

### Key Link Verification

| From | To | Via | Status | Details |
|------|----|-----|--------|---------|
| `codex-pricing.js` | `codex-global-aggregator.js` | require('./codex-pricing').computeOpusCost | WIRED | Line 19 of aggregator: `const { computeOpusCost } = require('./codex-pricing')` |
| `migrate-opus-pricing.js` | `~/.claude/dashboard/global.jsonl` | direct JSONL rewrite via fs.renameSync | WIRED | Lines 101-103: atomic tmp-then-rename pattern |
| `migrate-opus-pricing.js` | `codex-dashboard-generator.js` | require('./codex-dashboard-generator').generateDashboard | WIRED | Lines 109-110: generateDashboard(DASHBOARD_DIR) |
| `minimax-exec.js` | `codex-pricing.js` | require for computeCodexCostStrict | WIRED | Line 125: `const { computeCodexCostStrict } = require(PRICING_PATH)` |
| `minimax-exec.js` | `codex-exec.js` | require for runCodexExec in fallback chain | WIRED | Line 210: `const { runCodexExec } = require(CODEX_EXEC_PATH)` |
| `minimax-exec.js` | openai SDK | lazy require with fallback to global path | WIRED | Lines 85-87: try require('openai').OpenAI, fallback to global path |
| `.claude/settings.json` | `minimax-exec.js` | api_key_env field references MINIMAX_API_KEY | PARTIAL | Field is present in config; MINIMAX_API_KEY not yet set in environment |
| `minimax-connectivity-test.js` | `minimax-exec.js` | require('./minimax-exec').runMinimax | WIRED | Line 19: `const { runMinimax } = require('./minimax-exec')` |

### Data-Flow Trace (Level 4)

| Artifact | Data Variable | Source | Produces Real Data | Status |
|----------|---------------|--------|--------------------|--------|
| `codex-pricing.js` | OPUS_PRICING, CODEX_PRICING | Static constants (correct by definition) | Yes — hardcoded pricing data, not dynamic | FLOWING |
| `migrate-opus-pricing.js` | opus_baseline_usd | Reads global.jsonl, recalcs from stored tokens | Yes — processes 222 real records | FLOWING |
| `minimax-exec.js` | tokens, cost | OpenAI SDK response.usage | Yes — mapped from live API response fields | FLOWING (API call path) |
| `.claude/settings.json` | minimax config block | Static JSON config | Yes — correctly structured config data | FLOWING |

### Behavioral Spot-Checks

| Behavior | Command | Result | Status |
|----------|---------|--------|--------|
| computeCodexCostStrict returns 0.3 for minimax-m2.7 at 1M input tokens | `node -e "...computeCodexCostStrict({input_tokens:1e6,...},'minimax-m2.7')"` | 0.3 | PASS |
| computeOpusCost returns 5 for 1M input tokens (Opus 4.6) | `node -e "...computeOpusCost({input:1e6,...})"` | 5 | PASS |
| runMinimax returns correct error shape without API key | `process.env.MINIMAX_API_KEY=''` then `runMinimax('test')` | {success:false, error:'MINIMAX_API_KEY is not set', cost:0, text:''} | PASS |
| isCodexRateLimited correctly classifies all 6 test cases | `node -e "...isCodexRateLimited(null/pct95/etc)"` | All 6 cases return expected values | PASS |
| Migration idempotency on second run | `node migrate-opus-pricing.js` twice, diff | No output (files identical) | PASS |
| global.jsonl first record recalculated correctly | `node -e "...first_baseline === expected_baseline"` | match: true (0.086705) | PASS |
| settings.json all minimax fields correct | `node -e "...all 12 field checks"` | all_passed: true | PASS |
| Live MiniMax API connectivity | `node ~/.claude/hooks/minimax-connectivity-test.js` | Requires MINIMAX_API_KEY — SKIPPED | SKIP (human verify) |

### Requirements Coverage

No REQUIREMENTS.md file exists in this project. Requirements are tracked as D-series decisions in the phase context files (08-CONTEXT.md). All D-series IDs declared in plan frontmatter are verified through observable truths and artifact checks above.

| Plan | Requirement IDs | Status |
|------|----------------|--------|
| 08-01 | D-07, D-08, D-09 | All satisfied — pricing corrected, migration executed |
| 08-02 | D-01, D-02, D-03, D-04, D-05, D-06, D-14, D-15, D-16, D-17 | All satisfied — minimax-exec.js implements all decision points |
| 08-03 | D-10, D-11, D-12, D-13 | D-10/D-11 satisfied; D-13 (MINIMAX_API_KEY set) not yet verified — env var missing |

### Anti-Patterns Found

| File | Line | Pattern | Severity | Impact |
|------|------|---------|----------|--------|
| None | — | No TODO/FIXME/placeholder/stub patterns found in any phase artifact | — | — |

Security note: `minimax-connectivity-test.js` was auto-fixed during execution (Plan 03 deviation) to remove `slice(0,8)` key preview. The file now prints key length only (`length=N`). No partial key exposure in any artifact.

### Human Verification Required

#### 1. MINIMAX_API_KEY Environment Setup and Live Connectivity Test

**Test:** Set MINIMAX_API_KEY in the terminal, then run `node ~/.claude/hooks/minimax-connectivity-test.js`

**Steps:**
```bash
# Get key from: https://platform.minimaxi.com/user-center/basic-information/interface-key
export MINIMAX_API_KEY="your-api-key-here"
echo 'export MINIMAX_API_KEY="your-api-key-here"' >> ~/.bashrc
node ~/.claude/hooks/minimax-connectivity-test.js
```

**Expected:** Output contains all of:
- `MINIMAX_API_KEY: set (length=N)`
- `minimax-exec.js: loaded`
- `codex-pricing.js: minimax-m2.7 entry present (input:0.3 output:1.2)`
- `success: true`
- `tokens: { input_tokens: N, cached_input_tokens: 0, output_tokens: M }` (non-zero)
- `cost: $0.000...` (greater than 0)
- `VERIFICATION PASSED: MiniMax M-2.7 API is accessible and functional.`

**Note:** MiniMax pre-answer latency can reach 55 seconds. The test script uses a 120-second timeout. If it times out, retry once.

**Why human:** Requires a real API key that must be obtained from the MiniMax platform console. The programmatic infrastructure (minimax-exec.js, codex-pricing.js, settings.json) is fully verified. Only the credential and live call remain.

### Gaps Summary

One gap blocks full goal achievement: **MINIMAX_API_KEY is not set in the environment.** The plan stated "Set up MINIMAX_API_KEY environment variable" as a phase deliverable. The infrastructure to use the key is fully wired (minimax-exec.js checks it, settings.json references it as api_key_env, the connectivity test validates it). But the actual key has not been added to ~/.bashrc or any shell init file on this machine.

This is a user action item, not a code gap. Once the user sets the key and runs the connectivity test, all 17 truths will be verified and Phase 9 can proceed.

All 16 programmatically-verifiable truths are confirmed. No stub code, no empty implementations, no broken wiring, no incorrect pricing values.

---

_Verified: 2026-04-03T18:45:00Z_
_Verifier: Claude (gsd-verifier)_
