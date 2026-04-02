# Phase 3: Plan Review Loop & Superpowers - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md — this log preserves the alternatives considered.

**Date:** 2026-04-02
**Phase:** 03-plan-review-loop-superpowers
**Areas discussed:** Review loop dynamics, Superpowers routing, Loop visibility

---

## Review Loop Dynamics

### Q1: How should the review rounds flow between Opus and Codex?

| Option | Description | Selected |
|--------|-------------|----------|
| Opus drafts, Codex critiques (Recommended) | Opus writes plan → Codex reviews and suggests → Opus revises. | |
| Alternating proposals | Opus drafts → Codex counter-proposes → Opus picks best. | |
| Codex drafts, Opus reviews | Codex writes plan cheaply → Opus reviews for quality. | |

**User's choice:** Custom flow (Other): "Opus drafts → Codex critiques → Opus revises → Codex adversarial round on the final plan to poke holes and edge cases → Opus final revision"
**Notes:** User specifically wants two distinct Codex review types: constructive first, then adversarial. The adversarial round pokes holes and finds edge cases rather than just repeating the constructive review.

### Q2: Should the loop ever exit early if Codex has no issues?

| Option | Description | Selected |
|--------|-------------|----------|
| Yes — skip adversarial if clean (Recommended) | If first review finds zero issues, skip adversarial round. Saves time/tokens. | ✓ |
| No — always run both rounds | Both reviews always run regardless. Maximum thoroughness. | |
| You decide | Claude determines based on plan complexity. | |

**User's choice:** Yes — skip adversarial if clean (Recommended)
**Notes:** None

### Q3: When Opus and Codex disagree after all rounds, who wins?

| Option | Description | Selected |
|--------|-------------|----------|
| Opus always wins (Recommended) | Opus has final authority. Unresolved concerns go to 'decisions_not_taken'. | ✓ |
| Escalate to user | User breaks the tie if they still disagree. | |
| You decide | Claude picks based on disagreement severity. | |

**User's choice:** Opus always wins (Recommended)
**Notes:** None

---

## Superpowers Routing

### Q1: Which Superpowers parallel agents should route to GPT-5.4-mini?

| Option | Description | Selected |
|--------|-------------|----------|
| Hypothesis testing (Recommended) | Quick 'does this approach work?' checks. | ✓ |
| Code review threads | Parallel review passes on different files. | ✓ |
| Verification checks | Confirming implementation matches spec. | ✓ |
| None — keep all on Opus | Don't route to mini. No cost savings. | |

**User's choice:** All three selected (hypothesis testing, code review threads, verification checks)
**Notes:** None

### Q2: Should there be a fallback if GPT-5.4-mini gives low-confidence results?

| Option | Description | Selected |
|--------|-------------|----------|
| Yes — escalate to Opus (Recommended) | Low-confidence responses re-run on Opus. | ✓ |
| No — accept mini results | Whatever mini returns is final. | |
| You decide | Claude determines when escalation is warranted. | |

**User's choice:** Yes — escalate to Opus (Recommended)
**Notes:** None

---

## Loop Visibility

### Q1: How much of the review loop should you see?

| Option | Description | Selected |
|--------|-------------|----------|
| Milestone updates (Recommended) | Progress indicators: 'Round 1: Codex reviewing...' then '3 suggestions. Opus revising...' then final. | ✓ |
| Final result only | Nothing until loop finishes. Then final plan + summary. | |
| Full transparency | Every round's full output. Most verbose. | |
| You decide | Claude picks based on what keeps user informed. | |

**User's choice:** Milestone updates (Recommended)
**Notes:** None

### Q2: Should the final plan show what Codex changed vs the original draft?

| Option | Description | Selected |
|--------|-------------|----------|
| Yes — summary of changes (Recommended) | Plan includes section showing what review improved. User sees review value. | ✓ |
| No — just the final plan | Polished plan with no history. | |
| You decide | Include summaries when significant improvements made. | |

**User's choice:** Yes — summary of changes (Recommended)
**Notes:** None

---

## Skipped Areas

### Handoff Spec Content
Not selected for discussion. Left to Claude's discretion — Claude will design the typed handoff spec format based on downstream execution needs.

## Claude's Discretion

- Handoff spec format and `decisions_not_taken` structure
- Review state persistence implementation
- Adversarial review prompt design
- Low-confidence detection signals
- Superpowers skill modification approach

## Deferred Ideas

None — discussion stayed within phase scope.
