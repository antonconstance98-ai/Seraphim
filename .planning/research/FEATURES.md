# Feature Research

**Domain:** Idea-to-shipped workflow system (plugin extension)
**Researched:** 2026-04-09
**Confidence:** HIGH (domain analysis from existing Seraphim codebase + established PM/workflow patterns)

## Feature Landscape

### Table Stakes (Users Expect These)

Features users assume exist. Missing these = product feels incomplete.

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| Seed/Idea Capture | Braindumps go somewhere they won't be lost; thinking tools universally have capture | LOW | Structured freeform entry, timestamped, tagged. Stored in seeds.jsonl. Merge from Thought Orphanage. |
| Idea-to-roadmap promotion | Raw ideas must be promotable to actionable milestones; capture without promotion is a dead end | MEDIUM | `/seraphim:seed promote` — interrogate idea, generate REQ candidates, emit to roadmap |
| Requirements with IDs | Every PM tool that scales uses REQ-IDs; users cannot track "feature 7" without IDs | MEDIUM | REQ-001 format; scoped v1/future/out-of-scope; stored in requirements.jsonl. Depends on seed capture |
| Phased roadmap (waves not flat list) | Flat feature lists don't encode order, dependency, or grouping. Waves make sequencing explicit | MEDIUM | Wave 1/2/3 structure in roadmap.json; each wave has features + success criteria; upgrades existing milestone-feature hierarchy |
| Done criteria per task | Without done criteria, "done" is ambiguous; users expect it in any PM tool | LOW | Each plan task gets a `done_when` field; gates `/seraphim:done` |
| Progress visualization | Users need to see where they are; completion % is baseline expectation | MEDIUM | Bars + % in dashboard; depends on wave structure and done criteria |
| Human task enrichment | Decision/research/review tasks already exist (v3.1) but lack skill-level tagging and thought-task types | LOW | Extend existing HumanTask schema: add `skills`, `thought_prompt`, `leverage` fields |

### Differentiators (Competitive Advantage)

Features that set the product apart. Not required, but valuable.

| Feature | Value Proposition | Complexity | Notes |
|---------|-------------------|------------|-------|
| Research System with human interrogation first | AI research without scoping wastes tokens on the wrong questions; human interrogation before AI research focuses the search | MEDIUM | `/seraphim:research` interrogates human (5 questions), then spawns targeted research subagents. Categorized output: STACK/FEATURES/ARCHITECTURE/PITFALLS |
| Discuss phase / decision locking | Decisions made mid-build are harder to change; locking decisions before planning prevents expensive rework | MEDIUM | `/seraphim:discuss` runs structured debate, emits locked decisions to decisions.jsonl with immutable flag; planning reads locked decisions |
| Goal-backward verification | Most tools verify forward ("did we build X?"); goal-backward asks "does what we built achieve the original goal?" | HIGH | `/seraphim:verify` traces from delivered tasks back to REQ-IDs back to seed goals; surfaces drift |
| Wave-structured PLAN.md | Most planning tools produce flat task lists; wave structure makes parallelism and blocking relationships explicit | MEDIUM | PLAN.md sections: Wave N header, tasks with blockers/done_when, acceptance criteria per wave |
| Velocity tracking | Completion rate over time catches slowdowns before they become crises | MEDIUM | Rolling 7-day task completion rate in dashboard; requires timestamped done events in existing timeline.jsonl |
| REQ traceability matrix | Knowing which tasks satisfy which requirements catches under-implemented requirements before verification | HIGH | requirements.jsonl links REQ-IDs to plan tasks; dashboard shows coverage heatmap |
| Dashboard as control center | Passive metrics dashboard vs. active control center that lets you launch phases, mark tasks done, promote ideas | HIGH | Click-to-action in dashboard: promote seed, start wave, mark task done — calls plugin commands via fetch to a local socket |

### Anti-Features (Commonly Requested, Often Problematic)

| Feature | Why Requested | Why Problematic | Alternative |
|---------|---------------|-----------------|-------------|
| Fully automated idea-to-plan (no human checkpoints) | Feels fast; "AI does everything" | Requirements drift; AI interprets ambiguous seeds incorrectly; locked-in wrong direction before human sees output | Mandatory human interrogation gates at seed capture and discuss phase. Automation inside phases, human approval at phase boundaries |
| Per-task time estimates | Users want to know "how long"; sprints need estimates | Estimates are almost always wrong; they create false confidence and anchor bias on timelines | Use velocity (actual completion rate) to project finish; don't estimate upfront |
| Multi-project roadmap merging | Cross-project planning feels powerful | Different projects have different contexts; merged views obscure critical path per project; already have cross-project overview in v3.1 | Keep roadmaps per-project; cross-project overview stays read-only aggregate |
| AI-generated requirements (no human review) | Faster requirements definition | AI generates plausible-sounding but wrong-scoped requirements; REQ-IDs become meaningless | AI suggests requirement candidates from seed; human accepts/rejects/edits before IDs are assigned |
| Real-time collaborative editing | Multi-user feels enterprise-grade | This is a single-human plugin for a terminal workflow; collaboration adds conflict resolution complexity for zero benefit | Single-user, sequential editing. Add this if the project ever becomes multi-user |

## Feature Dependencies

```
Seed Capture
    └──enables──> Idea Promotion
                      └──enables──> Requirements Definition (REQ-IDs)
                                        └──enables──> Phased Roadmap (waves)
                                                          └──enables──> Wave-Structured PLAN.md
                                                                            └──enables──> Done Criteria Gates
                                                                                              └──enables──> Progress Visualization
                                                                                              └──enables──> Velocity Tracking
                                        └──enables──> REQ Traceability Matrix

Research System ──informs──> Requirements Definition
Discuss Phase ──locks-decisions-for──> Wave-Structured PLAN.md
Goal-Backward Verification ──requires──> REQ-IDs + Done Criteria Gates

Human Task Enrichment ──extends──> Existing HumanTask schema (v3.1)
Progress Visualization ──extends──> Existing Dashboard (v3.1)
Dashboard Control Center ──extends──> Existing Dashboard (v3.1)
```

### Dependency Notes

- **Seed Capture must precede Requirements Definition:** REQ-IDs need source material; without capture there is nothing to formalize.
- **Requirements must precede Phased Roadmap:** Wave success criteria reference REQ-IDs; flat roadmaps already exist in v3.1 so this upgrades the schema.
- **Discuss Phase locks before Planning:** PLAN.md generation reads locked decisions from decisions.jsonl; if discuss has not run, planning uses unlocked assumptions.
- **Done Criteria are prerequisite for Verification:** Goal-backward verification checks done_when against REQ-IDs; without done criteria the traceability chain is broken.
- **Progress Visualization is additive over existing dashboard:** No new backend needed; extends existing Neon schema + dashboard HTML.
- **Human Task Enrichment has no blockers:** It is a schema extension to existing HumanTask type; can ship independently.
- **Dashboard Control Center conflicts with static HTML approach:** Current dashboard is a static file; interactive actions require a local socket listener. High complexity — defer to end of milestone.

## MVP Definition

### Launch With (v3.2 core — P1 features)

Minimum set to deliver a coherent idea-to-shipped loop.

- [ ] Seed Capture (`/seraphim:seed`) — without this the idea-to-shipped loop has no entry point
- [ ] Requirements Definition (`/seraphim:requirements`) — REQ-IDs are the backbone of traceability
- [ ] Phased Roadmap upgrade (waves + success criteria) — extends roadmap.json; makes planning meaningful
- [ ] Wave-Structured PLAN.md generation (`/seraphim:plan`) — the planning output humans act on
- [ ] Progress Visualization in dashboard — completion bars + % are minimum feedback for the system

### Add After Core Loop Validated (v3.2 extended — P2 features)

- [ ] Research System (`/seraphim:research`) — human interrogation gate; valuable but not blocking the core loop
- [ ] Discuss Phase (`/seraphim:discuss`) — decision locking before planning; add once planning is working
- [ ] Enriched Human Tasks (skills/thought-prompt fields) — extends existing inbox; low effort, add once core ships
- [ ] Velocity Tracking — requires enough done events to be meaningful; add after a few tasks complete

### Future Consideration (v3.3+ — P3 features)

- [ ] Goal-Backward Verification (`/seraphim:verify`) — HIGH complexity; needs REQ traceability matrix first; high value but can ship without it
- [ ] REQ Traceability Matrix — substantial schema + UI work; defer until REQ-IDs have been in use
- [ ] Dashboard Control Center (click-to-action) — requires local socket listener; architectural change to static dashboard; high effort

## Feature Prioritization Matrix

| Feature | User Value | Implementation Cost | Priority |
|---------|------------|---------------------|----------|
| Seed Capture | HIGH | LOW | P1 |
| Requirements Definition (REQ-IDs) | HIGH | MEDIUM | P1 |
| Phased Roadmap (waves) | HIGH | MEDIUM | P1 |
| Wave-Structured PLAN.md | HIGH | MEDIUM | P1 |
| Progress Visualization | HIGH | MEDIUM | P1 |
| Research System | HIGH | MEDIUM | P2 |
| Discuss Phase | HIGH | MEDIUM | P2 |
| Human Task Enrichment | MEDIUM | LOW | P2 |
| Velocity Tracking | MEDIUM | MEDIUM | P2 |
| Goal-Backward Verification | HIGH | HIGH | P3 |
| REQ Traceability Matrix | MEDIUM | HIGH | P3 |
| Dashboard Control Center | HIGH | HIGH | P3 |

**Priority key:**
- P1: Must have for launch
- P2: Should have, add when possible
- P3: Nice to have, future consideration

## Competitor Feature Analysis

| Feature | Linear/Shortcut (PM tools) | GSD plugin (predecessor) | Seraphim v3.2 Approach |
|---------|---------------------------|--------------------------|------------------------|
| Idea capture | Inbox/backlog with manual promotion | No native capture | Structured seed capture with braindump + human interrogation at promotion |
| Requirements | Tickets with descriptions; no formal REQ-IDs | No requirements layer | REQ-IDs with v1/future/out-of-scope scoping; AI suggests, human approves |
| Phased planning | Cycles/sprints (time-boxed) | Four-phase workflow (task-based) | Wave-structured (dependency/readiness-based, not time-boxed) |
| Research | External (Notion, docs) | No native research | Integrated research subagent with human-scoped questions first |
| Verification | Manual acceptance testing | Crucible phase (code review) | Goal-backward: traces tasks to REQs to original goals |
| Progress | Burn-down charts, sprint velocity | No visualization | Completion bars, wave progress, velocity in existing dashboard |
| Decision tracking | Comments/history | decisions.jsonl (adaptive intel) | Discuss phase with explicit locking; locked decisions are immutable |

## Existing Seraphim Integration Points

Features in v3.2 that extend — not replace — existing v3.1 infrastructure:

| New Feature | Extends | How |
|-------------|---------|-----|
| Seed Capture | Existing inbox (human tasks) | New command + seeds.jsonl; separate from inbox but promotable |
| Requirements | roadmap.json | New requirements.jsonl; roadmap features reference REQ-IDs |
| Phased Roadmap | roadmap.json milestone-feature hierarchy | Add wave grouping + success_criteria fields to existing schema |
| Progress Visualization | Dashboard HTML + Neon sync | New bars/sparklines in existing PM panels |
| Human Task Enrichment | HumanTask schema in /seraphim:inbox | Add fields; backward-compatible |
| Velocity Tracking | timeline.jsonl (already timestamped) | Aggregate existing done events; new dashboard widget only |
| Discuss Phase | decisions.jsonl (adaptive intel) | New command; adds `locked: true` flag to decision entries |

## Sources

- Seraphim PROJECT.md — existing feature inventory and constraints
- v3.1 codebase — existing schema (roadmap.json, decisions.jsonl, HumanTask types)
- Established PM workflow patterns (Linear, Shortcut, GitHub Projects) — table stakes analysis
- GSD plugin predecessor analysis — what was carried forward vs. left behind
- Domain knowledge: idea-to-shipped workflows in AI-assisted development tools

---
*Feature research for: idea-to-shipped workflow capabilities (Seraphim v3.2)*
*Researched: 2026-04-09*
