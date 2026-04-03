---
phase: 05-data-pipeline
verified: 2026-04-03T01:10:00Z
status: passed
score: 4/4 success criteria verified
re_verification:
  previous_status: gaps_found
  previous_score: 3/4
  gaps_closed:
    - "A second consecutive run with no new data completes in under 5 ms and adds zero records"
  gaps_remaining: []
  regressions: []
---

# Phase 5: Data Pipeline Verification Report

**Phase Goal:** The global aggregator runs standalone, correctly merges all per-project token logs into `global.jsonl`, handles null session IDs as "unattributed", and completes a repeat run in under 5 ms
**Verified:** 2026-04-03T01:10:00Z
**Status:** passed
**Re-verification:** Yes — after gap closure via plan 05-03 (TTL-gated discovery cache)

## Goal Achievement

### Observable Truths (from ROADMAP Success Criteria)

| # | Truth | Status | Evidence |
|---|-------|--------|----------|
| 1 | Running the aggregator produces `global.jsonl` with records from all known projects, no duplicates | VERIFIED | 54 records across 4 projects; dedup Set verified; both Claude_X_Codex and gsd-workspaces present |
| 2 | Records with `session_id: null` appear in `global.jsonl` rather than being missing | VERIFIED | Null-session records confirmed in global.jsonl; dedup key = "null|timestamp" handles them correctly; `session_id: null` preserved in enriched record |
| 3 | A second consecutive run completes in under 5 ms and adds zero records | VERIFIED | `elapsed_ms: 2`, `records_added: 0` on warm run (measured); 2ms is well under the 5ms ROADMAP target; repeat run also 1ms |
| 4 | Discovery roots are read from `config.json` with sensible defaults so no projects are silently skipped | VERIFIED | `config.json` at `~/.claude/dashboard/config.json` with `extra_roots: []`, `max_depth: 5`; DEFAULT_ROOTS covers ~/projects, ~/agent, ~/gsd-workspaces, /mnt/hdd |

**Score:** 4/4 success criteria verified

---

## Gap Closure: 5 ms Repeat-Run Target

**Gap from previous verification:** `elapsed_ms: 151` on every no-op run due to 4x `spawnSync find` in `discoverTokenLogs()` being called unconditionally.

**Fix implemented (plan 05-03):** TTL-gated discovery cache in `codex-global-aggregator.js`.

**New symbols added:**
- `DISCOVERY_CACHE_TTL_MS = 60 * 60 * 1000` (1-hour TTL constant) — confirmed present (2 occurrences)
- `isCacheWarm()` — checks `project-index.json` mtime vs TTL before any subprocess spawn (3 occurrences)
- `loadDiscoveryCache()` — reads `discovered_files` array from `project-index.json` (2 occurrences)
- `wasWarm` flag in `aggregate()` — captured before any writes; gates warm/cold branch (4 occurrences)
- `discovered_files` top-level key in `project-index.json` — written on cold runs, read back on warm runs (9 occurrences)

**Measured warm-run timings:**
- Immediately after cold run: `elapsed_ms: 2` — PASS
- Second consecutive warm call: `elapsed_ms: 1` — PASS
- Both: `records_added: 0` — PASS (idempotent)

**Cold run regression check:**
- After `rm -f ~/.claude/dashboard/project-index.json`: exits 0, `discovered_files` array written with 4 entries, `global.jsonl` record count unchanged (54), hook-mode JSON output correct.

---

## Required Artifacts

### Plan 05-01 Artifacts (unchanged — no regression)

| Artifact | Expected | Status | Details |
|----------|----------|--------|---------|
| `~/.claude/hooks/codex-pricing.js` | Centralized pricing for all Codex and Opus models; 5 exports | VERIFIED | 106 lines; exports: CODEX_PRICING, OPUS_PRICING, computeCost, computeCodexCostStrict, computeOpusCost |
| `~/.claude/hooks/codex-exec.js` | Imports computeCost from codex-pricing.js; re-exports it | VERIFIED | Line 19: `require('./codex-pricing')`; re-export confirmed |
| `~/.claude/hooks/codex-cost-reporter.js` | Imports computeOpusCost from codex-pricing.js | VERIFIED | Line 16: `require('./codex-pricing')` with OPUS_PRICING, computeOpusCost |

### Plan 05-02 Artifacts (no regression)

| Artifact | Expected | Status | Details |
|----------|----------|--------|---------|
| `~/.claude/hooks/codex-global-aggregator.js` | Global aggregator with discovery, dedup, incremental reads | VERIFIED | 444 lines (grew from 384); exports `aggregate`; dual-mode; all new cache symbols present |
| `~/.claude/dashboard/global.jsonl` | Merged, deduplicated records from all projects | VERIFIED | 54 records (grew from 49 — 5 new real records added since plan 02 verification); no duplicates |
| `~/.claude/dashboard/project-index.json` | Discovery manifest with per-project metadata | VERIFIED | 4 projects; now also includes top-level `discovered_files` array with 4 entries |
| `~/.claude/dashboard/last-run.json` | Idempotency guard with run metadata | VERIFIED | Has `records_added`, `total_records`, `elapsed_ms`, `timestamp`, `projects_scanned` |
| `~/.claude/dashboard/cache.json` | Incremental read state — mtime and byte offset per source file | VERIFIED | 4 entries; each has `mtime_ms`, `size`, `byte_offset`; byte_offset = size (fast path active) |
| `~/.claude/dashboard/config.json` | User-configurable discovery roots and depth settings | VERIFIED | `extra_roots: []`, `max_depth: 5` |

### Plan 05-03 Artifact (gap closure)

| Artifact | Expected | Status | Details |
|----------|----------|--------|---------|
| `~/.claude/hooks/codex-global-aggregator.js` | Adds `DISCOVERY_CACHE_TTL_MS`, `isCacheWarm()`, `loadDiscoveryCache()`, warm/cold branch | VERIFIED | All 4 additions confirmed by grep; syntax check passes; warm run 2ms |

---

## Key Link Verification

### Plan 05-01 Key Links (no regression)

| From | To | Via | Status | Details |
|------|-----|-----|--------|---------|
| `codex-exec.js` | `codex-pricing.js` | `require('./codex-pricing')` | WIRED | Destructures `computeCost` and `CODEX_PRICING` |
| `codex-token-logger.js` | `codex-exec.js` | `const { computeCost } = require('./codex-exec')` | WIRED | Same function reference confirmed in plan 02 verification |
| `codex-cost-reporter.js` | `codex-pricing.js` | `require('./codex-pricing')` | WIRED | Destructures `OPUS_PRICING` and `computeOpusCost` |

### Plan 05-02 Key Links (no regression)

| From | To | Via | Status | Details |
|------|-----|-----|--------|---------|
| `codex-global-aggregator.js` | `codex-pricing.js` | `require('./codex-pricing')` | WIRED | Line 19: `const { computeOpusCost } = require('./codex-pricing')` |
| `codex-global-aggregator.js` | `~/.claude/dashboard/global.jsonl` | `fs.appendFileSync` | WIRED | Line 351: `fs.appendFileSync(GLOBAL_JSONL, ...)` |
| `codex-global-aggregator.js` | `~/.claude/dashboard/config.json` | `JSON.parse(fs.readFileSync(CONFIG_PATH))` | WIRED | `loadConfig()` reads or creates config.json |

### Plan 05-03 Key Links (gap closure)

| From | To | Via | Status | Details |
|------|-----|-----|--------|---------|
| `aggregate()` | `discoverTokenLogs()` | `wasWarm` flag — called only when `isCacheWarm()` returns false | WIRED | Lines 310-320: `wasWarm = isCacheWarm(); if (wasWarm) { loadDiscoveryCache() } else { discoverTokenLogs(...) }` |
| `project-index.json` | `discovered_files` array | `loadDiscoveryCache()` reads on warm runs | WIRED | `discovered_files` present in file (4 entries); `loadDiscoveryCache()` returns it when `Array.isArray(data.discovered_files)` is true |

---

## Data-Flow Trace (Level 4) — No regression from plan 02

| Artifact | Data Variable | Source | Produces Real Data | Status |
|----------|---------------|--------|-------------------|--------|
| `global.jsonl` | enriched records | `processFile()` reads source token-log.jsonl files | Yes — reads live project files | FLOWING |
| `global.jsonl`.`opus_baseline_usd` | `computeOpusCost(rec.tokens)` | `codex-pricing.js` OPUS_PRICING × real token counts | Yes | FLOWING |
| `global.jsonl`.`cost_usd` | `rec.cost_usd` | Copied from source token-log.jsonl as-is | Yes | FLOWING |
| `project-index.json` | `record_count` per project | Accumulated from `newRecords.length` per file | Yes | FLOWING |
| `project-index.json` | `discovered_files` | Written from `entries` on cold run; carried forward on warm run | Yes — real file paths from spawnSync find | FLOWING |

---

## Behavioral Spot-Checks

| Behavior | Result | Status |
|----------|--------|--------|
| `node --check codex-global-aggregator.js` | syntax OK | PASS |
| `DISCOVERY_CACHE_TTL_MS` count in file | 2 | PASS (definition + use in isCacheWarm) |
| `isCacheWarm` count in file | 3 | PASS (definition + call in aggregate + JSDoc) |
| `wasWarm` count in file | 4 | PASS (declaration + if branches + Step 7) |
| Cold run after `rm -f project-index.json` | exit 0, 4 projects discovered, 54 records unchanged | PASS |
| `discovered_files` in project-index.json after cold run | array with 4 entries, valid filePath fields | PASS |
| Warm run `elapsed_ms` | 2ms | PASS (<5ms target) |
| Warm run `records_added` | 0 | PASS (idempotent) |
| Second warm run `elapsed_ms` | 1ms | PASS |
| Second warm run `records_added` | 0 | PASS |
| Hook mode `echo '{...}' \| node aggregator.js` | `{"additionalContext":"Dashboard: 0 new records aggregated (54 total across 4 projects)"}` exit 0 | PASS |
| `module.exports = { aggregate }` at end of file | Present | PASS |
| `global.jsonl` record count unchanged after warm run | 54 = 54 | PASS |

**Note on records_added=1 on first warm-run check:** The first `aggregate()` call during re-verification picked up 1 genuinely new record written to the Claude_X_Codex token-log since the previous aggregation. This is correct behavior — the source file had grown. A subsequent call returned `records_added: 0`, confirming idempotency. The byte_offset in `cache.json` matched `token-log.jsonl` size after that run, confirming the fast path is active for subsequent calls.

---

## Requirements Coverage

| Requirement | Source Plan | Description | Status | Evidence |
|-------------|------------|-------------|--------|----------|
| PIPE-01 | 05-02-PLAN.md | Global aggregator scans all projects and merges into global.jsonl with deduplication | SATISFIED | 54 records from 4 projects; 0 duplicates; `appendFileSync` wired to global.jsonl; REQUIREMENTS.md checked [x] |
| PIPE-02 | 05-01-PLAN.md, 05-02-PLAN.md | Discovery roots configurable via config.json with sensible defaults | SATISFIED | config.json has `extra_roots` and `max_depth`; DEFAULT_ROOTS covers all CLAUDE.md key paths; REQUIREMENTS.md checked [x] |
| PIPE-03 | 05-02-PLAN.md, 05-03-PLAN.md | Aggregator uses mtime-gated incremental reads to skip unchanged files | SATISFIED | `processFile()` fast path via mtime+size check; plan 05-03 extends this with TTL-gated discovery cache; warm run 2ms; REQUIREMENTS.md checked [x] |
| PIPE-04 | 05-02-PLAN.md | Records with null session_id tracked as "unattributed" rather than dropped | SATISFIED | `buildDedupKey` produces "null|timestamp"; enriched record preserves `session_id: null`; confirmed in global.jsonl; REQUIREMENTS.md checked [x] |

All 4 PIPE requirements satisfied. All marked `[x]` in REQUIREMENTS.md. No orphaned requirements.

**Orphaned requirements check:** REQUIREMENTS.md maps PIPE-01 through PIPE-04 to Phase 5. All 4 are claimed across 05-01, 05-02, and 05-03 plans. No orphaned requirements.

---

## Anti-Patterns Found

| File | Line | Pattern | Severity | Impact |
|------|------|---------|----------|--------|
| `codex-global-aggregator.js` | 165, 168 | `return {}` in `loadCache()` | Info | Intentional: returns empty cache on first run or corrupt cache. Causes full read on first invocation, which is correct. |

No blockers or warnings. No new anti-patterns introduced by plan 05-03.

---

## Human Verification Required

None. The 5ms target is now verifiably met by automated measurement (2ms). The human clarification item from the initial verification is resolved by the gap closure — the implementation now meets the ROADMAP criterion as written.

---

## Summary

**All 4 ROADMAP success criteria are now verified.**

The single gap from the initial verification — the 5ms repeat-run target — is closed. Plan 05-03 added a TTL-gated discovery cache that skips the 4x `spawnSync find` child processes on warm runs. The warm-run elapsed_ms dropped from 151ms to 2ms (30x improvement), well under the 5ms ROADMAP target. Cold runs still discover all projects and write `discovered_files` into `project-index.json` for the next warm run.

The gap closure introduced no regressions: all plan 05-01 and 05-02 key links remain wired, all 4 PIPE requirements remain satisfied, global.jsonl record count is intact, and hook mode produces correct `additionalContext` JSON.

Phase 5 is complete.

---

_Verified: 2026-04-03T01:10:00Z_
_Verifier: Claude (gsd-verifier)_
