---
phase: 05-data-pipeline
verified: 2026-04-03T00:29:15Z
status: gaps_found
score: 3/4 success criteria verified
gaps:
  - truth: "A second consecutive run with no new data completes in under 5 ms and adds zero records"
    status: partial
    reason: "Records added is 0 (idempotency correct) but elapsed_ms is 151ms, not under 5ms. The 5ms target in the ROADMAP success criterion applies to the full aggregate() wall time. The implementation includes 4x spawnSync find invocations for project discovery (~150ms). The file-I/O fast path itself takes <1ms (verified). The plan body acknowledges this trade-off and acceptance criteria omit the 5ms constraint, but the ROADMAP goal and Success Criterion 3 state the full run must be under 5ms."
    artifacts:
      - path: "~/.claude/hooks/codex-global-aggregator.js"
        issue: "discoverTokenLogs() runs 4x spawnSync find on every aggregate() call (lines 105-112), contributing ~150ms overhead regardless of whether any files changed. A lazy/cached discovery path (re-run find only if discovery cache is stale) or replacing spawnSync with a glob-based stat walk would bring the total elapsed_ms under 5ms."
    missing:
      - "Mtime-gated discovery cache: skip the spawnSync find calls when no directory mtimes have changed since last run, OR cache discovered file paths in project-index.json and only re-run find when no cache exists"
      - "Alternative: use fs.readdirSync recursive walk instead of spawnSync find to avoid child_process overhead"
human_verification:
  - test: "Confirm ROADMAP success criterion intent for 5ms"
    expected: "Verify whether '5ms' was meant to cover the full run (including discovery) or only the incremental file-read fast path — if the intent was the latter, this gap is a doc/spec mismatch rather than a code defect"
    why_human: "The plan body explicitly carves out discovery latency ('5ms target applies to the incremental read logic, not the full aggregate() including discovery') but the ROADMAP success criterion says 'completes in under 5ms' without qualification. Only the project owner can clarify the authoritative target."
---

# Phase 5: Data Pipeline Verification Report

**Phase Goal:** The global aggregator runs standalone, correctly merges all per-project token logs into `global.jsonl`, handles null session IDs as "unattributed", and completes a repeat run in under 5 ms
**Verified:** 2026-04-03T00:29:15Z
**Status:** gaps_found
**Re-verification:** No — initial verification

## Goal Achievement

### Observable Truths (from ROADMAP Success Criteria)

| # | Truth | Status | Evidence |
|---|-------|--------|----------|
| 1 | Running the aggregator produces `global.jsonl` with records from all known projects, no duplicates | VERIFIED | 49 records across 4 projects; dedup Set verified (49 unique keys = 49 total); both Claude_X_Codex and The-Crucible present |
| 2 | Records with `session_id: null` appear in `global.jsonl` rather than being missing | VERIFIED | 6 null-session records confirmed in global.jsonl; `session_id: null` field preserved as-is; dedup key = "null|timestamp" handles them correctly |
| 3 | A second consecutive run completes in under 5 ms and adds zero records | PARTIAL | `records_added: 0` confirmed (idempotency correct); `elapsed_ms: 151` ms on every no-op run — 30x the 5ms target. File-I/O fast path is <1ms but discovery via 4x spawnSync find adds ~150ms. |
| 4 | Discovery roots are read from `config.json` with sensible defaults so no projects are silently skipped | VERIFIED | `config.json` exists at `~/.claude/dashboard/config.json` with `extra_roots: []` and `max_depth: 5`; DEFAULT_ROOTS covers all CLAUDE.md key paths (~/projects, ~/agent, ~/gsd-workspaces, /mnt/hdd); extra_roots merges at runtime |

**Score:** 3/4 success criteria verified (1 partial)

---

## Required Artifacts

### Plan 05-01 Artifacts

| Artifact | Expected | Status | Details |
|----------|----------|--------|---------|
| `~/.claude/hooks/codex-pricing.js` | Centralized pricing for all Codex and Opus models; 5 exports | VERIFIED | 106 lines; exports: CODEX_PRICING, OPUS_PRICING, computeCost, computeCodexCostStrict, computeOpusCost; no shebang (library module) |
| `~/.claude/hooks/codex-exec.js` | Imports computeCost from codex-pricing.js; re-exports it | VERIFIED | Line 19: `const { CODEX_PRICING, computeCost } = require('./codex-pricing')`; line 278: `module.exports = { runCodexExec, parseCodexTokens, computeCost, runGpt54MiniApi }` — re-export confirmed |
| `~/.claude/hooks/codex-cost-reporter.js` | Imports computeOpusCost from codex-pricing.js | VERIFIED | Line 16: `const { OPUS_PRICING, computeOpusCost } = require('./codex-pricing')`; inline constants removed |

### Plan 05-02 Artifacts

| Artifact | Expected | Status | Details |
|----------|----------|--------|---------|
| `~/.claude/hooks/codex-global-aggregator.js` | Global aggregator with discovery, dedup, incremental reads | VERIFIED | 384 lines (>100 required); exports `aggregate`; dual-mode; imports computeOpusCost from codex-pricing.js |
| `~/.claude/dashboard/global.jsonl` | Merged, deduplicated records from all projects | VERIFIED | 49 records; all have `project_name`, `project_path`, `opus_baseline_usd` (number); 6 null-session records; no duplicate dedup keys |
| `~/.claude/dashboard/project-index.json` | Discovery manifest with per-project metadata | VERIFIED | 4 projects; record_count fields match actual global.jsonl counts (25/20/3/1) |
| `~/.claude/dashboard/last-run.json` | Idempotency guard with run metadata | VERIFIED | Has `records_added`, `total_records`, `elapsed_ms`, `timestamp`, `projects_scanned` |
| `~/.claude/dashboard/cache.json` | Incremental read state — mtime and byte offset per source file | VERIFIED | 4 entries; each has `mtime_ms`, `size`, `byte_offset`; byte_offset = size (fast path ready) |
| `~/.claude/dashboard/config.json` | User-configurable discovery roots and depth settings | VERIFIED | `extra_roots: []`, `max_depth: 5` |

---

## Key Link Verification

### Plan 05-01 Key Links

| From | To | Via | Status | Details |
|------|-----|-----|--------|---------|
| `codex-exec.js` | `codex-pricing.js` | `require('./codex-pricing')` | WIRED | Line 19: destructures `computeCost` and `CODEX_PRICING` |
| `codex-token-logger.js` | `codex-exec.js` | `const { computeCost } = require('./codex-exec')` | WIRED | Line 59 confirmed; `exec.computeCost === pricing.computeCost` is `true` (same function reference) |
| `codex-cost-reporter.js` | `codex-pricing.js` | `require('./codex-pricing')` | WIRED | Line 16: destructures `OPUS_PRICING` and `computeOpusCost` |

### Plan 05-02 Key Links

| From | To | Via | Status | Details |
|------|-----|-----|--------|---------|
| `codex-global-aggregator.js` | `codex-pricing.js` | `require('./codex-pricing')` | WIRED | Line 19: `const { computeOpusCost } = require('./codex-pricing')` |
| `codex-global-aggregator.js` | `~/.claude/dashboard/global.jsonl` | `fs.appendFileSync` | WIRED | Line 306: `fs.appendFileSync(GLOBAL_JSONL, JSON.stringify(enriched) + '\n', 'utf8')` |
| `codex-global-aggregator.js` | `~/.claude/dashboard/config.json` | `JSON.parse(fs.readFileSync(CONFIG_PATH))` | WIRED | Lines 84/81: reads or creates config.json via `loadConfig()` |

---

## Data-Flow Trace (Level 4)

| Artifact | Data Variable | Source | Produces Real Data | Status |
|----------|---------------|--------|-------------------|--------|
| `global.jsonl` | enriched records | `processFile()` reads source token-log.jsonl files | Yes — reads live project files with real token/cost data | FLOWING |
| `global.jsonl`.`opus_baseline_usd` | `computeOpusCost(rec.tokens)` | `codex-pricing.js` OPUS_PRICING constants × actual token counts | Yes — computed from real token counts from source records | FLOWING |
| `global.jsonl`.`cost_usd` | `rec.cost_usd` | Copied as-is from source token-log.jsonl (written at logging time by codex-token-logger.js) | Yes — real values from prior sessions | FLOWING |
| `project-index.json` | `record_count` per project | Accumulated from `newRecords.length` per file processed | Yes — counts match global.jsonl (verified: 25/20/3/1) | FLOWING |
| `cache.json` | `byte_offset` | Set to `stat.size` after each file read | Yes — offsets match actual file sizes | FLOWING |

---

## Behavioral Spot-Checks

| Behavior | Result | Status |
|----------|--------|--------|
| `node codex-global-aggregator.js` exits 0 (hook mode via pipe) | `{"additionalContext":"Dashboard: 0 new records aggregated (49 total across 4 projects)"}` | PASS |
| `aggregate()` exported function returns correct shape | `{ projects_scanned:4, records_added:0, total_records:49, elapsed_ms:167 }` | PASS |
| Idempotency: records_added=0 on repeat run | Confirmed — wc -l before=49, after=49; `records_added: 0` in last-run.json | PASS |
| `exec.computeCost === pricing.computeCost` (same function reference) | `true` | PASS |
| `computeCodexCostStrict({input_tokens:1000000}, 'fake')` returns null | `null` | PASS |
| `computeCost({input_tokens:1000000}, 'fake')` returns number (fallback) | `2.5` | PASS |
| Null session_id records present in global.jsonl | 6 records with `session_id: null` | PASS |
| No duplicate dedup keys in global.jsonl | 49 keys, 49 unique — no duplicates | PASS |
| `elapsed_ms` under 5ms on no-op run | 151ms — FAILS 5ms ROADMAP target | FAIL |
| File-I/O fast path (dedup + stat only, no find) | <1ms | PASS |

---

## Requirements Coverage

| Requirement | Source Plan | Description | Status | Evidence |
|-------------|------------|-------------|--------|----------|
| PIPE-01 | 05-02-PLAN.md | Global aggregator scans all projects and merges into global.jsonl with deduplication | SATISFIED | 49 records from 4 projects; 0 duplicates; `appendFileSync` wired to global.jsonl |
| PIPE-02 | 05-01-PLAN.md, 05-02-PLAN.md | Discovery roots configurable via config.json with sensible defaults | SATISFIED | config.json has `extra_roots` and `max_depth`; DEFAULT_ROOTS covers ~/projects, ~/agent, ~/gsd-workspaces, /mnt/hdd |
| PIPE-03 | 05-02-PLAN.md | Aggregator uses mtime-gated incremental reads to skip unchanged files | SATISFIED | `processFile()` fast path: returns early when `mtime_ms === currentMtime && size === currentSize`; byte_offset advanced to file size after read; file-I/O on no-op run <1ms |
| PIPE-04 | 05-02-PLAN.md | Records with null session_id tracked as "unattributed" rather than dropped | SATISFIED | `buildDedupKey` produces "null|timestamp" string; enriched record preserves `session_id: null`; 6 null records confirmed in global.jsonl |

All 4 PIPE requirements are satisfied at the implementation level. The 5ms gap is a ROADMAP goal metric, not a separate requirement.

**Orphaned requirements check:** REQUIREMENTS.md maps PIPE-01 through PIPE-04 to Phase 5. All 4 are claimed across 05-01 and 05-02 plans. No orphaned requirements.

---

## Anti-Patterns Found

| File | Line | Pattern | Severity | Impact |
|------|------|---------|----------|--------|
| `codex-global-aggregator.js` | 161, 164 | `return {}` in `loadCache()` | Info | Not a stub — intentional: returns empty cache on first run or if cache file is corrupt/missing. The empty dict causes all files to be fully read on first invocation, which is correct behavior. |

No blockers or warnings found. The `return {}` instances are legitimate empty-state defaults, not hollow implementations.

---

## The 5ms Gap: Analysis

**ROADMAP Success Criterion 3 states:** "A second consecutive run with no new data completes in under 5 ms and adds zero new records to global.jsonl"

**Observed:** `elapsed_ms: 151` on every no-op run.

**Root cause:** `discoverTokenLogs()` runs 4 separate `spawnSync('find', [...])` invocations on every call — one per discovery root (`~/projects`, `~/agent`, `~/gsd-workspaces`, `/mnt/hdd`). Each `find` invocation has child-process spawn overhead. Measured: ~150ms total for 4 roots.

**What is fast:** The actual file-I/O fast path (loading global.jsonl into dedup Set, stat'ing cached files, finding no changes) takes <1ms — well under the 5ms target.

**Plan's clarification:** The plan body (Task 2 action section) explicitly states: "the 5 ms target applies to the incremental read logic, not the full aggregate() including discovery. If discovery adds latency, that is acceptable as long as records_added: 0 and no file I/O beyond stat() occurs."

**Assessment:** The incremental read fast path meets the spirit of the 5ms requirement. However, the ROADMAP success criterion as written does not distinguish between discovery and file-read time — it says the full run must be under 5ms. Since the criterion is the authoritative contract and the measured time is 30x the target, this is classified as a gap requiring either a fix or an explicit scope revision to the ROADMAP.

**Fix options (in order of simplicity):**
1. Cache discovered paths in `project-index.json`; only re-run `find` when `project-index.json` does not exist or is older than N minutes. Expected elapsed_ms on no-op: <2ms.
2. Replace `spawnSync find` with a JavaScript recursive `fs.readdirSync` walk (eliminates child-process overhead; ~5-10ms for shallow walks).
3. Update ROADMAP Success Criterion 3 to read "...completes with zero new file I/O (mtime fast path active)..." if the intent was only the file-read portion.

---

## Human Verification Required

### 1. Clarify 5ms target scope

**Test:** Review ROADMAP.md Phase 5 Success Criterion 3 and confirm: does "completes in under 5 ms" include project discovery (the find commands), or was it intended to cover only the incremental file-read fast path?

**Expected:** Either (a) the target includes discovery, in which case the fix options above apply, or (b) the target was meant for file-I/O only, in which case the ROADMAP criterion should be updated to reflect the actual design.

**Why human:** The plan body and the ROADMAP criterion are in direct tension. Only the project owner can determine which is authoritative.

---

## Gaps Summary

One gap blocks full achievement of the phase goal as written:

**5ms repeat-run target not met.** The ROADMAP success criterion requires a no-op run to complete in under 5ms. Actual elapsed_ms is ~151ms due to project discovery (4x spawnSync find). The incremental file-read logic itself is fast (<1ms). All other success criteria are verified: idempotency (records_added=0), correct global.jsonl with all 4 projects and 49 records, null session preservation, and configurable discovery roots. The gap is narrow and fixable by caching discovered paths between runs.

---

_Verified: 2026-04-03T00:29:15Z_
_Verifier: Claude (gsd-verifier)_
