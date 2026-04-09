---
phase: 01-core-pm-primitives
verified: 2026-04-09T00:00:00Z
status: passed
score: 18/18 must-haves verified
re_verification: false
---

# Phase 01: Core PM Primitives Verification Report

**Phase Goal:** Developer can create roadmaps, queue features, start features through the pipeline, view human tasks, and close milestones -- all from terminal commands, with PM context surviving pause/resume
**Verified:** 2026-04-09
**Status:** passed
**Re-verification:** No — initial verification

## Goal Achievement

### Observable Truths

| # | Truth | Status | Evidence |
|---|-------|--------|----------|
| 1 | readRoadmap() returns { milestones: [] } when roadmap.json missing | VERIFIED | Live node test confirmed output `{"milestones":[]}` |
| 2 | Roadmap data survives concurrent writes via atomic temp-file rename | VERIFIED | roadmap.js line 34-36: writes to `roadmap.json.tmp`, then `fs.renameSync` |
| 3 | findFeature() resolves both feat-NNN IDs and slugs | VERIFIED | Live node test: slug lookup returns `feat-001`, ID lookup returns `test-slug` |
| 4 | /seraphim:roadmap displays milestone-feature tree with status icons and progress | VERIFIED | roadmap.md lines 57-64: icon map `[●][✓][ ][!]`; `milestoneProgress` called; `--all` flag handled |
| 5 | Feature costs attributed in milestone archival by feature_id | VERIFIED | close-milestone.md lines 88-113: filters decisions.jsonl by `featureIds.includes(record.feature_id)` |
| 6 | config.js CONFIG_DEFAULTS includes max_wip: 2 | VERIFIED | Live node test confirmed `max_wip: 2` |
| 7 | /seraphim:add-feature prompts interactively for name, milestone, description, priority | VERIFIED | add-feature.md has `--name`, `--milestone`, `--slug`, `--description`, `--priority` flags with interactive fallback prompts |
| 8 | /seraphim:start moves feature to in-progress and warns if WIP limit exceeded | VERIFIED | start.md: sets `feature.status = 'in-progress'`, WIP warning emitted when `currentWip >= maxWip` (warns, not blocks) |
| 9 | Feature reordering possible via direct JSON edit | VERIFIED | add-feature.md line 11 explicitly documents reordering via direct `.seraphim/roadmap.json` edit (QUEUE-04 minimal) |
| 10 | Neither add-feature nor start blocks on missing roadmap.json | VERIFIED | Both commands call `readRoadmap()` which returns `{ milestones: [] }` on missing file |
| 11 | No sprint/story-point/time-boxing concepts in any command | VERIFIED | add-feature.md line 130 comment: "no sprint, no story points, no time estimates — ARCH-06/D-10" |
| 12 | /seraphim:inbox groups tasks by project then by type | VERIFIED | inbox.md lines 100-132: groups by `projectName` then iterates `VALID_TYPES` |
| 13 | /seraphim:inbox shows 5 task types: decision, research, review, validation, skills | VERIFIED | inbox.md line 102: `VALID_TYPES = ['decision', 'research', 'review', 'validation', 'skills']` |
| 14 | /seraphim:done marks task complete via sidecar JSONL, not forge-log mutation | VERIFIED | done.md line 114: `fs.appendFileSync(completionsPath, ...)` to `task-completions.jsonl`; no forge-log writes |
| 15 | pause.md writes pm block to state.json with feature_id, milestone_version, progress | VERIFIED | pause.md lines 72-84: `state.pm = { feature_id, milestone_version, progress }` |
| 16 | resume.md reads pm block from state.json and restores PM context | VERIFIED | resume.md lines 57-66: reads `state.pm`, prints feature_id and milestone_version if present |
| 17 | close-milestone.md freezes milestone to .seraphim/milestones/vX.Y.json with cost | VERIFIED | close-milestone.md: atomic write to `milestones/{version}.json`, removes from active roadmap |
| 18 | Pipeline gates emit SERAPHIM:HUMAN_TASKS markers; feature auto-completes on SHIP | VERIFIED | run.md lines 248-386: Step 6e emits markers at pre-envision/pre-architect/post-crucible; Step 7b sets `feature.status = 'complete'` on SHIP verdict |

**Score:** 18/18 truths verified

### Required Artifacts

| Artifact | Expected | Status | Details |
|----------|----------|--------|---------|
| `~/.claude/plugins/seraphim/lib/roadmap.js` | 6 exported functions for roadmap CRUD | VERIFIED | All 6 exports confirmed: readRoadmap, writeRoadmap, findFeature, nextFeatureId, countWip, milestoneProgress |
| `~/.claude/plugins/seraphim/lib/config.js` | max_wip: 2 in CONFIG_DEFAULTS | VERIFIED | Live test: `max_wip: 2` |
| `~/.claude/plugins/seraphim/lib/decisions-logger.js` | nullable feature_id in buildRecord | VERIFIED | Live test: default null, accepts `feat-001` |
| `~/.claude/plugins/seraphim/commands/roadmap.md` | Roadmap display command | VERIFIED | Status icons, milestoneProgress, --all flag, empty-state message |
| `~/.claude/plugins/seraphim/commands/add-feature.md` | Interactive feature creation | VERIFIED | writeRoadmap, nextFeatureId, `status: 'planned'`, `depends_on: []`, no anti-features |
| `~/.claude/plugins/seraphim/commands/start.md` | Feature start with WIP warning | VERIFIED | findFeature, countWip, max_wip comparison, delegates to /seraphim:run |
| `~/.claude/plugins/seraphim/commands/inbox.md` | Unified human task inbox | VERIFIED | parseMarkers, task-completions.jsonl filter, 5 VALID_TYPES, grouped by project+type |
| `~/.claude/plugins/seraphim/commands/done.md` | Task completion command | VERIFIED | appendFileSync to task-completions.jsonl, completed_at timestamp |
| `~/.claude/plugins/seraphim/commands/pause.md` | Extended pause with PM context | VERIFIED | state.pm block with feature_id, milestone_version, progress |
| `~/.claude/plugins/seraphim/commands/resume.md` | Extended resume reading PM context | VERIFIED | state.pm read; backward compat when absent |
| `~/.claude/plugins/seraphim/commands/close-milestone.md` | Milestone archival command | VERIFIED | decisions.jsonl cost attribution, atomic archive write, roadmap cleanup |
| `~/.claude/plugins/seraphim/commands/run.md` | Pipeline with markers and auto-complete | VERIFIED | Step 6e gates + Step 7b auto-complete on SHIP |

### Key Link Verification

| From | To | Via | Status | Details |
|------|----|-----|--------|---------|
| roadmap.md | roadmap.js | require in node -e block | WIRED | Line 46: `require('${PLUGIN_ROOT}/lib/roadmap')` |
| roadmap.js readRoadmap | .seraphim/roadmap.json | fs.readFileSync | WIRED | Confirmed via live test returning correct empty structure |
| add-feature.md | roadmap.js writeRoadmap, nextFeatureId | require | WIRED | Both functions called in add-feature.md lines 128, 152 |
| start.md | roadmap.js findFeature, countWip | require | WIRED | Lines 63, 87 confirmed |
| start.md | config.js max_wip | require | WIRED | Line 86: `cfg.max_wip || 2` |
| inbox.md | markers.js parseMarkers | require | WIRED | Line 40 |
| inbox.md | task-completions.jsonl | fs.readFileSync | WIRED | Line 58 |
| done.md | task-completions.jsonl | fs.appendFileSync | WIRED | Line 114 |
| pause.md Step 3 | state.json | state.pm assignment | WIRED | Line 84: `state.pm = pmBlock` |
| resume.md Step 3 | state.json | reads state.pm | WIRED | Lines 57-66 |
| close-milestone.md | .seraphim/milestones/ | writeFileSync + renameSync | WIRED | Line 138 |
| close-milestone.md | decisions.jsonl | readFileSync + filter by feature_id | WIRED | Lines 89-103 |
| run.md Step 6e | forge-log.md | emitMarker HUMAN_TASKS | WIRED | Lines 292-300 |
| run.md Step 7b | roadmap.js writeRoadmap | node -e script | WIRED | Lines 346, 386 |

### Data-Flow Trace (Level 4)

These are slash command instruction files (markdown), not components rendering dynamic data. Data flows at command execution time, not at file-read time. Level 4 not applicable.

### Behavioral Spot-Checks

| Behavior | Command | Result | Status |
|----------|---------|--------|--------|
| readRoadmap returns empty on missing file | `node -e "require(...).readRoadmap('/tmp/nonexistent_xyz')"` | `{"milestones":[]}` | PASS |
| findFeature resolves slug | `node -e "...findFeature({...}, 'test-slug')"` | `feat-001` | PASS |
| findFeature resolves feat-NNN ID | `node -e "...findFeature({...}, 'feat-001')"` | `test-slug` | PASS |
| config max_wip = 2 | `node -e "require(...).CONFIG_DEFAULTS.max_wip"` | `2` | PASS |
| decisions-logger feature_id defaults null | `node -e "...buildRecord({...}).feature_id"` | `null` | PASS |
| decisions-logger feature_id accepts value | `node -e "...buildRecord({..., feature_id:'feat-001'}).feature_id"` | `feat-001` | PASS |

### Requirements Coverage

| Requirement | Source Plan | Description | Status | Evidence |
|-------------|------------|-------------|--------|---------|
| ROAD-01 | 01-01 | Roadmap stored in `.seraphim/roadmap.json` with milestone-feature hierarchy | SATISFIED | roadmap.js readRoadmap/writeRoadmap confirmed; schema with milestones array |
| ROAD-02 | 01-01 | `/seraphim:roadmap` displays milestone-feature tree with statuses | SATISFIED | roadmap.md renders status icons, progress, handles --all flag |
| ROAD-03 | 01-02, 01-05 | `/seraphim:add-feature` appends feature to milestone backlog | SATISFIED | add-feature.md writes to roadmap.json with planned status |
| ROAD-04 | 01-04 | Milestone completion freezes to `.seraphim/milestones/vX.Y.json` | SATISFIED | close-milestone.md atomic write confirmed |
| ROAD-05 | 01-01 | Milestone progress percentage from feature statuses | SATISFIED | milestoneProgress() live-tested: `{complete:2,total:3,percent:67}` |
| ROAD-07 | 01-04 | Milestone cost tracking aggregates decisions.jsonl per feature | SATISFIED | close-milestone.md filters by feature_id, sums cost_usd |
| QUEUE-01 | 01-01, 01-02 | Feature backlog with `planned` status | SATISFIED | add-feature.md sets `status: 'planned'` on new features |
| QUEUE-02 | 01-02 | `/seraphim:start` moves feature to in-progress and launches pipeline | SATISFIED | start.md: status updated to `in-progress`, delegates to /seraphim:run |
| QUEUE-03 | 01-02 | WIP limit (default 2) enforced with warn on `/seraphim:start` | SATISFIED | start.md: countWip vs max_wip comparison, warning not block |
| QUEUE-04 | 01-02 | Feature reordering via command or direct JSON edit | SATISFIED | add-feature.md documents direct JSON edit; --priority flag for insertion |
| TASK-01 | 01-03 | `/seraphim:inbox` aggregates pending human tasks across active features | SATISFIED | inbox.md scans in-progress features' forge-log.md files |
| TASK-02 | 01-03 | 5 human task types: decision, research, review, validation, skills | SATISFIED | VALID_TYPES array in inbox.md and done.md |
| TASK-03 | 01-03 | `/seraphim:done` marks task complete without re-running pipeline | SATISFIED | done.md appends to task-completions.jsonl sidecar only |
| TASK-04 | 01-03, 01-05 | Pipeline gates write human tasks visible in inbox | SATISFIED | run.md Step 6e emits HUMAN_TASKS markers; inbox.md reads them via parseMarkers |
| ARCH-01 | 01-01 | PM layer is read-path only, never gates pipeline | SATISFIED | run.md: marker emission failure is silent; PM writes are advisory-only |
| ARCH-02 | 01-01 | decisions-logger.js extended with nullable feature_id | SATISFIED | Live test: `feature_id: null` default, accepts value |
| ARCH-03 | 01-04 | pause.md state.json extended with PM context block | SATISFIED | pause.md `state.pm = { feature_id, milestone_version, progress }` |
| ARCH-06 | 01-02 | No sprint/story-points/time-boxing anti-features | SATISFIED | add-feature.md explicitly comments exclusion; no anti-feature terms found |

**Orphaned requirements:** None. All 18 requirement IDs from plan frontmatter cross-reference to verified implementations.

### Anti-Patterns Found

| File | Pattern | Severity | Impact |
|------|---------|----------|--------|
| None | — | — | No stubs, placeholders, or empty handlers found in any phase artifact |

Checked patterns across all 12 modified files: no `TODO/FIXME`, no `return null` in rendering paths, no hardcoded empty data passed to callers. All implementations are substantive.

### Human Verification Required

None. All observable truths for this phase are verifiable programmatically via grep and node execution. The commands are markdown instruction files, not UI components requiring visual inspection.

### Gaps Summary

No gaps. All 18 must-haves from plans 01-01 through 01-05 are verified against the actual codebase. Every artifact exists, is substantive, and is correctly wired to its dependencies. The phase goal — developer can create roadmaps, queue features, start features, view human tasks, and close milestones from terminal commands, with PM context surviving pause/resume — is fully achieved.

---

_Verified: 2026-04-09_
_Verifier: Claude (gsd-verifier)_
