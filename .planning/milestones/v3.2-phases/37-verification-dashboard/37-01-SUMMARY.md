---
phase: 37-verification-dashboard
plan: "01"
subsystem: seraphim-commands
tags: [verification, uat, nyquist, commands]
dependency_graph:
  requires: []
  provides: [verify.md, uat.md, validate.md]
  affects: [seraphim-plugin-commands]
tech_stack:
  added: []
  patterns: [atomic-tmp-renameSync, subagent-dispatch, yaml-frontmatter-parse]
key_files:
  created:
    - ~/.claude/plugins/seraphim/commands/verify.md
    - ~/.claude/plugins/seraphim/commands/uat.md
    - ~/.claude/plugins/seraphim/commands/validate.md
  modified: []
decisions:
  - "verify.md mandates REQUIRES_HUMAN_JUDGMENT in every report — verifier subagent prompt enforces this via explicit instruction"
  - "uat.md derives UAT items from VERIFICATION.md Observable Truths + Required Artifacts tables on first run"
  - "validate.md stops gracefully if VERIFICATION.md not found — correct dependency ordering enforced"
  - "All three commands use atomic tmp+renameSync write pattern consistent with debug.md/requirements.js"
metrics:
  duration: "8 minutes"
  completed: "2026-04-10T18:34:05Z"
  tasks_completed: 2
  files_created: 3
requirements: [VFY-01, VFY-02, VFY-03, VFY-04]
---

# Phase 37 Plan 01: Verification Commands Summary

**One-liner:** Three verification commands — goal-backward verify with REQ-ID traceability, conversational UAT with persistent YAML state, and nyquist-auditor gap analysis.

## What Was Built

Three command files added to `~/.claude/plugins/seraphim/commands/`:

### verify.md (VFY-01, VFY-02)
Goal-backward verification command. Reads all `*-PLAN.md` files in a phase directory, extracts `requirements:` array and `must_haves:` block (truths, artifacts, key_links) from YAML frontmatter, cross-references REQ-IDs against REQUIREMENTS.md, then spawns a verifier subagent. The subagent prompt explicitly mandates at least one `REQUIRES_HUMAN_JUDGMENT` item — identifying the most subjective quality aspect only a human can evaluate. Writes VERIFICATION.md with Observable Truths table, Required Artifacts table, and Key Link Verification table. Uses atomic tmp+renameSync write pattern.

### uat.md (VFY-04)
Conversational UAT command with persistent state. On first run, reads VERIFICATION.md and generates UAT.md with all truths and artifacts as pending items. On subsequent runs, reads existing UAT.md, finds the first pending item, presents it to the user, and records their passed/failed verdict with notes. YAML frontmatter tracks total/tested/passed/failed counts and overall status (in_progress / complete / has_failures). All writes use atomic tmp+renameSync pattern to prevent torn-write corruption across sessions.

### validate.md (VFY-03)
Nyquist gap auditing command. Reads VERIFICATION.md, then spawns a nyquist-auditor subagent that identifies four classes of coverage gaps: shallow evidence (file-existence checks that don't verify behavior), missing edge cases, unvalidated integration points, and REQUIRES_HUMAN_JUDGMENT items needing concrete UAT scenarios. Writes VALIDATION.md with severity-classified gaps (critical/important/minor). Stops with a helpful message if VERIFICATION.md not found — enforcing the verify → validate → uat dependency order.

## Deviations from Plan

None — plan executed exactly as written. All three commands follow the debug.md/forensics.md pattern with correct frontmatter, subagent dispatch, and atomic writes.

## Known Stubs

None — all commands are fully wired. The verify/uat/validate pipeline is complete end-to-end.

## Self-Check

Automated verification passed:
- `verify.md` exists and contains `REQUIRES_HUMAN_JUDGMENT`: PASS
- `uat.md` exists and contains `UAT.md` and `renameSync`: PASS
- `validate.md` exists and contains `nyquist-auditor`, `VALIDATION.md`, and `renameSync`: PASS

## Self-Check: PASSED
