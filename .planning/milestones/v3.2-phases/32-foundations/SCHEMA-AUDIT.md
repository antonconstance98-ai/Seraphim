# Schema Extension Audit — v3.2 Idea-to-Shipped Journey

**Date:** 2026-04-09
**Phase:** 32-foundations (FOUND-04)
**Purpose:** Confirm every v3.2 data concept maps to an existing structure or justify any new file. This document is the prerequisite gate for v3.2 feature work per D-10.

---

## Principle: Extend, Not Duplicate

Per D-10: every v3.2 data concept must extend an existing top-level data store unless the concept has a cross-feature, independent lifecycle that cannot be scoped to any single existing structure. Any new top-level data store must carry an explicit justification.

---

## Mapping Table

| v3.2 Concept | Existing Home | Extension Approach | New Store? | Justification |
|---|---|---|---|---|
| **Seeds** | `.planning/seeds/` directory | Add `index.jsonl` as lookup index; each seed is `SEED-NNN.md` file | No | Directory already exists from prior workflow. Seeds are inherently per-project ephemeral captures — extending the existing dir is correct. |
| **Requirements** | `.seraphim/roadmap.json` features array | Add `requirements[]` array with `req_id` and `scope` fields to each feature object | No | Requirements belong to features in the milestone-feature hierarchy. Extending roadmap.json keeps the structure self-contained. |
| **Waves** | `.seraphim/roadmap.json` milestones/features array | Add `waves[]` array to feature objects for wave-structured planning with dependencies | No | Waves are a planning overlay on top of the feature hierarchy — extending roadmap.json is the correct home. |
| **Discuss decisions** | `.planning/phases/NN-slug/NN-CONTEXT.md` | Already captured in CONTEXT.md per current GSD/Seraphim workflow decisions block | No new file | The existing CONTEXT.md pattern is the canonical decision capture mechanism. No duplication needed. |
| **Research items** | `.seraphim/research.json` (new file) | Follows roadmap.js atomic write pattern (`readResearch`/`writeResearch`) | Yes — justified | Research items are cross-feature with independent lifecycle. A research item may inform multiple features across milestones and persists independently of any one feature's status. Scoping to a single feature's roadmap entry would break cross-feature research links. |
| **Human task enrichment** | Existing `HumanTask` type | Add optional fields: `skills_to_learn: string`, `thought_prompt: string`, `research_task: string` | No | These are additive optional fields on the existing type. No new store required — the existing human task inbox handles persistence. |
| **Progress / velocity** | `.seraphim/timeline.jsonl` + `task-completions.jsonl` | Computed at read time from existing append-only logs | No | Progress metrics are derived, not persisted. Both source files already exist; computation is a read-time operation with no new persistence layer needed. |

---

## Summary

6 of 7 concepts extend existing structures. `research.json` is the only new file, justified by its cross-feature scope and independent lifecycle. This satisfies D-10.

---

## Verification Checklist

| Concept | Target File / Structure Exists Today? | Extension Needed |
|---|---|---|
| Seeds | Yes — `.planning/seeds/` directory exists | Add `index.jsonl` lookup index |
| Requirements | Yes — `.seraphim/roadmap.json` exists | Add `requirements[]` to feature objects |
| Waves | Yes — `.seraphim/roadmap.json` exists | Add `waves[]` to feature objects |
| Discuss decisions | Yes — CONTEXT.md pattern established in every phase | None — pattern already captures decisions |
| Research items | No — `.seraphim/research.json` does not exist | Create new file; follows roadmap.js atomic write pattern |
| Human task enrichment | Yes — `HumanTask` type in plugin lib | Add 3 optional fields to existing type |
| Progress / velocity | Yes — `timeline.jsonl` + `task-completions.jsonl` exist | None — compute at read time, no new persistence |
