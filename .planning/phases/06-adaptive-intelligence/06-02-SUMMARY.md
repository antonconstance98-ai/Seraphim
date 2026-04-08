---
phase: 06-adaptive-intelligence
plan: "02"
subsystem: adaptive-intelligence
tags: [pattern-analysis, recommendation-engine, jsonl, pure-logic]
dependency_graph:
  requires: [06-01]
  provides: [pattern-analyzer.js, recommendation-engine.js]
  affects: [06-03, 06-04]
tech_stack:
  added: []
  patterns: [CommonJS, Node.js stdlib only, append-only JSONL]
key_files:
  created:
    - ~/.claude/plugins/seraphim/lib/pattern-analyzer.js
    - ~/.claude/plugins/seraphim/lib/recommendation-engine.js
  modified: []
decisions:
  - "aggregateDecisions accepts both projectRoot string and records array — test scaffold passes array directly"
  - "computeProfileMetrics detects flat vs grouped input — accepts both shapes"
  - "confidence fallback derived from n when metric.confidence is absent — required for test compatibility"
key_decisions:
  - "aggregateDecisions accepts array or projectRoot — test scaffold passes array directly"
  - "confidence derived from n when metric lacks field — test passes metric without confidence key"
metrics:
  duration_minutes: 2
  completed_date: "2026-04-08"
  tasks_completed: 2
  files_created: 2
  files_modified: 0
requirements_satisfied: [ADPT-01, ADPT-02, ADPT-03]
---

# Phase 06 Plan 02: Pattern Analyzer and Recommendation Engine Summary

Pattern analysis and rule-based recommendation generation implemented as two pure-logic CommonJS modules using Node.js stdlib only.

## What Was Built

### pattern-analyzer.js

Four exported functions:

- `aggregateDecisions(input)` — accepts projectRoot string or records array; filters out meta-records (type=recommendation, type=recommendation_response)
- `groupIntoRuns(records)` — groups flat records into run sub-arrays; new discover record after prior records starts a new run
- `computeRejectionRates(records)` — groups by phase+model; failure = outcome=failure OR loop_count>0; confidence LOW/MEDIUM/HIGH by n<5/n<20/n>=20; sorted by rejection_rate descending
- `computeProfileMetrics(input)` — accepts flat records or pre-grouped runs; avg_cost per run count; avg_crucible_pass_rate from quality_signals field (null if no crucible data)

### recommendation-engine.js

Four exported items:

- `RULES` — two rules: high-rejection-rate (rate>=0.6 AND n>=3) and low-crucible-pass-rate (rate<0.5 AND n>=3)
- `generateRecommendations(metrics, existingRecords)` — fires rules against metrics; deduplicates against pending recommendations by rule_id+phase+model; generates rec_id as 8-char sha256 hex
- `appendRecommendations(projectRoot, recommendations)` — writes to decisions.jsonl via appendDecision; no-op on empty array
- `appendRejection(projectRoot, rec_id, reason)` — appends recommendation_response record; append-only, never modifies original

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 1 - Bug] confidence field absent on test metric**
- **Found during:** Task 2 — test-recommendation-engine.js Test 3
- **Issue:** Test passes `{ rejection_rate: 0.8, n: 5 }` without a confidence field; rule returned `undefined`
- **Fix:** confidence function derives from n when metric.confidence is absent: `m.confidence || (n<5?'LOW':n<20?'MEDIUM':'HIGH')`
- **Files modified:** recommendation-engine.js
- **Commit:** e1e59f5

**2. [Rule 1 - Bug] aggregateDecisions called with array in test**
- **Found during:** Task 1 — reading test-pattern-analyzer.js
- **Issue:** Plan spec describes projectRoot string input; test passes records array directly
- **Fix:** aggregateDecisions detects input type with Array.isArray — supports both
- **Files modified:** pattern-analyzer.js
- **Commit:** ce0d978

**3. [Rule 1 - Bug] computeProfileMetrics called with flat records in test**
- **Found during:** Task 1 — reading test-pattern-analyzer.js
- **Issue:** Plan spec says input is runs (array of arrays); test passes flat records
- **Fix:** computeProfileMetrics detects flat vs grouped input; calls groupIntoRuns if flat
- **Files modified:** pattern-analyzer.js
- **Commit:** ce0d978

## Known Stubs

None — both modules are pure logic with no UI rendering or placeholder data.

## Self-Check: PASSED

- pattern-analyzer.js: FOUND at ~/.claude/plugins/seraphim/lib/pattern-analyzer.js
- recommendation-engine.js: FOUND at ~/.claude/plugins/seraphim/lib/recommendation-engine.js
- SUMMARY.md: FOUND at .planning/phases/06-adaptive-intelligence/06-02-SUMMARY.md
- Commit ce0d978: FOUND (pattern-analyzer)
- Commit e1e59f5: FOUND (recommendation-engine)
