# Project Research Summary

**Project:** Seraphim v3.2 — Idea-to-Shipped Journey
**Domain:** Workflow orchestration layer on existing AI PM plugin
**Researched:** 2026-04-09
**Confidence:** HIGH

## Executive Summary

Seraphim v3.2 extends an already-working three-layer system (six-phase execution pipeline, PM layer, dashboard) with a pre-execution idea pipeline: seed capture, research, requirements definition, discussion/decision locking, and wave-structured planning. Every concept in v3.2 has a home in the existing codebase — waves extend milestones, REQ-IDs extend feature records, seeds are features with `status: seed`, verification is an extension of the Crucible phase. The primary risk is designing these as parallel systems rather than schema extensions.

Zero new npm dependencies needed. This is a schema and command extension project, not a new system build.

The critical risk is v3.1 technical debt: Neon DDL never applied, project vs project_name mismatch, feature_id not flowing through decisions logger. Phase 1 must be a debt-clearance gate.

## Key Findings

### Recommended Stack

No new dependencies. Node.js fs, roadmap.js patterns, Chart.js 4.5.1, Neon — all already installed.

### Expected Features

**Must have (P1):** Seed capture, REQ-IDs, wave-structured roadmaps, PLAN.md generation, progress visualization
**Should have (P2):** Research system (two-command), discuss phase, enriched human tasks, velocity tracking
**Defer (P3):** Goal-backward verification, REQ traceability matrix, dashboard click-to-action control center

### Architecture Approach

Additive vertical slice above existing pipeline. New lib files (requirements.js, wave-planner.js, research-tracker.js) follow roadmap.js pattern. New commands follow markdown-instruction + node -e pattern. Dashboard extends via existing push-client.

### Critical Pitfalls

1. Duplicating existing PM primitives — extend, don't parallel
2. Research skipping interrogation — must be two separate commands
3. Neon schema divergence from v3.1 debt — clear before any v3.2 work
4. Markdown commands accumulating logic — extract to Node.js libs
5. Verification as AI checkbox — require REQUIRES_HUMAN_JUDGMENT items

## Implications for Roadmap

6 phases suggested:
1. v3.1 Debt Clearance + Schema Audit (prerequisite gate)
2. Data Foundations — Libs + Core Commands (requirements, waves, seed, discuss, plan)
3. Research System (two-command architecture)
4. Progress Visualization + Dashboard (batch all UI)
5. Enriched Human Tasks + Velocity Tracking (low-risk extension)
6. Verification System (highest complexity, last)

## Confidence Assessment

| Area | Confidence |
|------|------------|
| Stack | HIGH |
| Features | HIGH |
| Architecture | HIGH |
| Pitfalls | HIGH (structural), MEDIUM (workflow) |

---
*Research completed: 2026-04-09*
*Ready for roadmap: yes*
