# Phase 32: Foundations - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md — this log preserves the alternatives considered.

**Date:** 2026-04-09
**Phase:** 32-foundations
**Areas discussed:** Neon DDL strategy, Schema unification, Extension audit format, feature_id wiring

---

## All Areas — Claude's Discretion

User selected "you decide everything" when presented with the four gray areas.

All decisions made by Claude based on codebase analysis:

| Decision | Rationale |
|----------|-----------|
| Migration SQL in `dashboard/migrations/`, idempotent | Matches repo-committed approach, safe to re-run |
| Standardize on `project_name` (snake_case) | Matches existing config.json and push-client conventions |
| ALTER TABLE RENAME for existing mismatches | Clean fix, one-time migration |
| feature_id from pm-context.js through call sites | Module already exposes active feature; minimal wiring |
| Audit doc at `SCHEMA-AUDIT.md` in phase dir | Consumed once then archived with phase |

## Claude's Discretion

All four areas were delegated to Claude:
- Neon DDL strategy
- Schema unification approach
- Schema extension audit format
- feature_id wiring approach

## Deferred Ideas

None
