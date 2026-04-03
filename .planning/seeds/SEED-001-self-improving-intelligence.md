---
id: SEED-001
status: dormant
planted: 2026-04-03
planted_during: v2.0 Three-Model Intelligence (post-completion)
trigger_when: after v2.0 has run for 1-2 weeks with real MiniMax data, or next major milestone
scope: Large
---

# SEED-001: Self-Improving Intelligence

Aggregate execution data (review gate false positives, verification gaps, fallback events, plan review accuracy) into a feedback loop that tunes prompts, thresholds, and routing decisions.

## Why This Matters

The three-model system (v2.0) generates rich execution data — token logs, review verdicts, verification scores, fallback events — but none of it feeds back to improve the system. Patterns repeat without being caught:

- **Review gate false positives:** During the v2.0 autonomous run, the Codex review gate produced 4 consecutive false positives because it pattern-matched conversation context (Phase 10 HANDOFF.md mentions of "blocked verdict" and "high severity") against the current response. No mechanism detected the pattern or adjusted the prompt.
- **Plan review truncation:** The Codex multi-round reviewer consistently flagged issues caused by plan truncation (plans exceeding the reviewer's context window), not real problems. Each planning phase wasted a review round on the same structural limitation.
- **Verification gaps vs real bugs:** Phase 13 verification found 2 real gaps (stdout pollution, API key echo), but Phase 8's gap was a user-action prerequisite (API key not set). No classification distinguishes "code defect" from "environment prerequisite."
- **Fallback chain unused:** The Codex-to-MiniMax fallback chain was built but never triggered during v2.0 because Codex never hit rate limits during the session. No data on real-world fallback frequency exists yet.

Without a feedback loop, the system can't learn from its own execution history. Prompts stay static, thresholds are guesses, and the same false positives recur every session.

## When to Surface

**Trigger:** After v2.0 has run for 1-2 weeks across multiple projects with real MiniMax data flowing through the pipeline.

This seed should be presented during `/gsd:new-milestone` when the milestone scope matches any of these conditions:
- Milestone involves "optimization", "tuning", or "improvement" of the hook system
- Milestone involves "metrics", "analytics", or "observability" beyond basic token tracking
- Milestone involves "ML", "learning", "adaptive", or "self-improving" capabilities
- At least 2 weeks of real three-model execution data exists in global.jsonl

## Scope Estimate

**Large** — This is a full milestone. Involves:
- Data collection schema changes (classify false positives, track review accuracy)
- Aggregation pipeline extension (pattern detection across sessions)
- Prompt tuning framework (A/B test different review gate prompts)
- Threshold auto-adjustment (compression thresholds, scan skip thresholds based on hit rates)
- Dashboard extensions (false positive rate, review accuracy, fallback frequency)
- Possibly a lightweight SQLite store replacing raw JSONL for queryable history

## Breadcrumbs

Related code and decisions in the current codebase:

- `~/.claude/hooks/codex-review-gate.js` — Stop hook that produced false positives; prompt needs scoping to response content only
- `~/.claude/hooks/codex-multi-round-reviewer.js` — Plan reviewer that hits truncation; needs plan size awareness
- `~/.claude/hooks/codex-token-logger.js` — v2.0 fields (dual_review, review_round, round_model, fallback_from, compression) are forward-compatible placeholders with no producers yet
- `~/.claude/hooks/codex-handoff.js` — Fallback chain that logs to token-log.jsonl; source of fallback event data
- `~/.claude/hooks/codex-dashboard-generator.js` — Fallback Events panel exists but will show empty until real data flows
- `~/.claude/dashboard/global.jsonl` — 222 records, all gpt-5.4; no MiniMax data yet
- `.planning/v2.0-MILESTONE-AUDIT.md` — Documents the integration blocker (stderr/stdout) and tech debt items
- `.planning/phases/10-adversarial-plan-review/10-HANDOFF.md` — Source of the review gate false positive context bleed

## Notes

Key insight from the v2.0 autonomous run: the review gate's prompt receives the full conversation context, not just the response being reviewed. This means earlier discussion of "blocked verdicts" and "high severity concerns" (from Phase 10 planning) causes the gate to flag clean responses as problematic. A quick win would be to scope the review prompt to only the most recent response content — but the deeper fix is a feedback loop that detects repeated false positives and adjusts automatically.

The v2.0 token-logger already has 5 unused v2.0 fields waiting for producers. These were designed as forward-compatible placeholders specifically for this kind of feedback data.
