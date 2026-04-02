# Phase 2: Review Gate & GSD Integration - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md — this log preserves the alternatives considered.

**Date:** 2026-04-02
**Phase:** 02-review-gate-gsd-integration
**Areas discussed:** Review gate trigger, Review feedback visibility, GSD checkpoint behavior, Review scope

---

## Review Gate Trigger

### Q1: When should the Stop hook trigger a Codex review?

| Option | Description | Selected |
|--------|-------------|----------|
| Code changes only (Recommended) | Only review when Claude's response includes Write/Edit/Bash that modified files. Skips chat-only responses. | ✓ |
| Every response | Review all Claude responses regardless of content. Most thorough but adds latency to every interaction. | |
| Configurable threshold | Review when changes exceed a threshold (e.g., 3+ files changed, or 50+ lines). | |
| You decide | Claude picks the right trigger logic. | |

**User's choice:** Code changes only (Recommended)
**Notes:** None

### Q2: Should the review gate be bypassable when iterating fast?

| Option | Description | Selected |
|--------|-------------|----------|
| Yes — quick toggle | A flag or command lets you disable the review gate temporarily. | |
| No — always on | If review gate is enabled for a project, it always runs. Consistency over speed. | ✓ |
| You decide | Claude picks based on hook architecture. | |

**User's choice:** No — always on
**Notes:** None

### Q3: What happens when Codex finds an issue?

| Option | Description | Selected |
|--------|-------------|----------|
| Block and fix (Recommended) | Codex blocks the response, Opus fixes the issue before user sees anything. | ✓ |
| Warn but deliver | User gets response plus a 'Codex flagged: [issue]' note. | |
| You decide | Claude picks based on severity. | |

**User's choice:** Block and fix (Recommended)
**Notes:** None

---

## Review Feedback Visibility

### Q1: How visible should the review process be?

| Option | Description | Selected |
|--------|-------------|----------|
| Summary only (Recommended) | One-line note: 'Codex reviewed: PASS' or 'Codex reviewed: fixed [issue]'. Minimal noise. | ✓ |
| Detailed feedback | Full review output — what was checked, found, fixed. | |
| Silent | Review happens behind the scenes. Cleanest but no visibility. | |
| You decide | Claude picks verbosity level. | |

**User's choice:** Summary only (Recommended)
**Notes:** None

### Q2: When Codex blocks and Opus fixes, should you see what was fixed?

| Option | Description | Selected |
|--------|-------------|----------|
| Yes — brief note | 'Codex caught: [issue]. Fixed before delivery.' | ✓ |
| No — just deliver | Corrected result with no mention of the fix. | |
| You decide | Claude picks based on fix significance. | |

**User's choice:** Yes — brief note
**Notes:** None

### Q3: Should review results be logged?

| Option | Description | Selected |
|--------|-------------|----------|
| Yes — append to token log (Recommended) | Review events in .planning/token-log.jsonl alongside token usage. | ✓ |
| Separate review log | Reviews in their own .planning/review-log.jsonl. | |
| No logging | Reviews are ephemeral. | |

**User's choice:** Yes — append to token log (Recommended)
**Notes:** None

---

## GSD Checkpoint Behavior

### Q1: At plan-phase finalization, should Codex review block or advise?

| Option | Description | Selected |
|--------|-------------|----------|
| Block until reviewed (Recommended) | Plan can't be finalized until Codex reviews. Feedback incorporated into plan file. | ✓ |
| Advisory only | Codex reviews, feedback shown, but plan proceeds regardless. | |
| You decide | Claude picks based on GSD plan-phase workflow. | |

**User's choice:** Block until reviewed (Recommended)
**Notes:** None

### Q2: At GSD wave boundaries, should validation be blocking or non-blocking?

| Option | Description | Selected |
|--------|-------------|----------|
| Non-blocking (Recommended) | Codex validates in background. Results surface at natural stopping points. | ✓ |
| Blocking | Execution pauses at each wave boundary until Codex validates. | |
| You decide | Claude picks based on GSD execution patterns. | |

**User's choice:** Non-blocking (Recommended)
**Notes:** None

### Q3: If wave validation finds a critical issue, should it halt the next wave?

| Option | Description | Selected |
|--------|-------------|----------|
| Yes — halt on critical (Recommended) | Non-blocking normally, but critical issues block the next wave. | ✓ |
| Never halt | Results always advisory. Execution never blocked. | |
| You decide | Claude determines what's "critical" and when to halt. | |

**User's choice:** Yes — halt on critical (Recommended)
**Notes:** None

---

## Review Scope

### Q1: What should Codex focus on when reviewing?

| Option | Description | Selected |
|--------|-------------|----------|
| Bugs and logic errors | Functional issues: wrong logic, off-by-one, null handling, race conditions. | ✓ |
| Security vulnerabilities | Injection risks, exposed secrets, unsafe patterns. | ✓ |
| Requirements alignment | Code implements what the plan/spec asked for. | ✓ |
| Style and conventions | Consistent naming, patterns, code organization. | ✓ |

**User's choice:** All four selected
**Notes:** None

### Q2: Should Codex review the diff only, or also read surrounding context?

| Option | Description | Selected |
|--------|-------------|----------|
| Diff + relevant context (Recommended) | Changed lines plus enough surrounding code to understand impact. | ✓ |
| Diff only | Only git diff. Cheapest but may miss context-dependent issues. | |
| Full files | Entire changed files. Most thorough but highest cost. | |
| You decide | Claude picks best quality-to-cost ratio. | |

**User's choice:** Diff + relevant context (Recommended)
**Notes:** None

### Q3: Should review depth vary by task type?

| Option | Description | Selected |
|--------|-------------|----------|
| Vary by task type (Recommended) | Deep for features/security. Light for tests/bulk ops. Optimizes cost vs quality. | ✓ |
| Same depth always | Equally thorough regardless of task type. | |
| You decide | Claude determines depth by risk level. | |

**User's choice:** Vary by task type (Recommended)
**Notes:** None

---

## Claude's Discretion

- Infinite loop prevention (`stop_hook_active` guard)
- Critical issue classification criteria
- Review prompt design for each review type
- GSD hook integration points (event names, matcher patterns)
- Global routing hook extension for non-GSD workflows

## Deferred Ideas

None — discussion stayed within phase scope.
