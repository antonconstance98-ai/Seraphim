---
status: partial
phase: 03-six-phase-pipeline-and-profile-management
source: [03-VERIFICATION.md]
started: 2026-04-08T22:00:00Z
updated: 2026-04-08T22:00:00Z
---

## Current Test

[awaiting human testing]

## Tests

### 1. Full pipeline run on Performance profile
expected: `/seraphim:run {N}` on Performance profile executes all six phases and writes all output files (external.md, internal.md, vision.md, judgment.md, blueprint.md, forge-log.md, crucible.md) to `.seraphim/phases/{N}/`
result: [pending]

### 2. Profile switch and override round-trip
expected: `/seraphim:set-profile balanced` prints assignment table; `/seraphim:override judge gemini-3.1-pro` changes only Judge assignment
result: [pending]

### 3. opus_enabled: false fallback rendering
expected: Setting `opus_enabled: false` causes Opus-assigned phases to show profile-specific fallback models in show-profile output
result: [pending]

## Summary

total: 3
passed: 0
issues: 0
pending: 3
skipped: 0
blocked: 0

## Gaps
