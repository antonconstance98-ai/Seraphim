# Domain Pitfalls: Project Management Layer on an AI Execution Pipeline

**Domain:** Adding project management (roadmaps, feature queues, progress tracking, cross-project oversight) to an existing AI pipeline plugin (Seraphim v3.0)
**Context:** Subsequent milestone — v3.0 shipped a working six-phase pipeline; v3.1 wraps it in PM infrastructure
**Researched:** 2026-04-08
**Confidence:** HIGH for second system effect and ceremony creep (well-documented in Brooks, confirmed by multiple sources); HIGH for file-based state pitfalls (verified against AI agent memory research and ACID literature); HIGH for GSD-specific integration pitfalls (observed directly from reading GSD source); MEDIUM for session continuity specifics (pattern observed in existing pause/resume commands, generalized)

---

## Critical Pitfalls

---

### Pitfall 1: Second System Effect — Building a PM Tool Instead of a PM Feature

**What goes wrong:**
v3.0 succeeded. It shipped 13 phases, 43 plans, 21 commands, and adaptive intelligence. That success generates confidence. v3.1 starts as "add roadmaps and feature queues" and ends up as a full Kanban implementation, sprint planning, velocity tracking, burndown charts, dependency graphs, and a standalone project management philosophy. The PM layer becomes bigger than the pipeline it was supposed to serve.

This is Fred Brooks' second-system effect verbatim: the first system earns its way in with humility; the second carries all the ambitions held back from the first. Every feature sounds reasonable in isolation. Collectively they constitute a rewrite.

**Why it happens in this specific context:**
Seraphim already has a dashboard (Vercel), JSONL logging, decision tracking, adaptive intelligence, and nine model integrations. The temptation is to connect everything to everything. "Since we have decisions.jsonl, let's surface it in the roadmap. Since we have token costs, let's show them per-feature. Since we have model performance data, let's factor it into milestone estimates." Each extension is technically easy. Together they create an unmaintainable surface area.

**Consequences:**
- v3.1 takes 3-4x longer than estimated
- The PM layer needs its own maintenance, documentation, and debugging
- Users (just the developer, in this case) avoid using it because setup overhead defeats the purpose
- The pipeline — which was the actual product — gets neglected during the PM build

**Prevention:**
- Define scope with explicit anti-features before building anything: "This version does NOT include sprint planning, velocity tracking, or automated dependency resolution"
- Each proposed PM feature must answer: "Does this make a six-phase pipeline run faster or better, or does it add ceremony for its own sake?"
- Use GSD's own planning infrastructure (ROADMAP.md, STATE.md) as the PM layer until it demonstrably fails — only build what GSD cannot do
- Cap v3.1 feature count at N before starting; any addition requires removing something else

**Detection (warning signs):**
- The feature list grows between Discover and Architect phases
- A new data model is proposed that doesn't map to an existing file (decisions.jsonl, token-log.jsonl, state.json)
- Estimated phase count for v3.1 exceeds the phase count for v3.0
- The first implementation question is "what schema should the roadmap use" rather than "what problem does the roadmap solve"

**Phase to address:** Roadmap creation / Discover phase. Scope lock must happen before any technical design. The roadmap for v3.1 must explicitly list what is OUT OF SCOPE as prominently as what is in scope.

---

### Pitfall 2: Ceremony Creep — PM Infrastructure Blocks the Pipeline It's Supposed to Serve

**What goes wrong:**
PM tools fail when they add overhead to the work they are tracking. The classic failure mode: a task cannot move to "In Progress" without a milestone assignment; a milestone cannot be created without a roadmap; a roadmap requires a project; a project requires a configuration wizard. By the time you can start work, you've spent 20 minutes filling out forms.

In Seraphim's context this manifests as: a feature cannot enter the six-phase pipeline without a roadmap entry; a roadmap entry requires a milestone; closing a milestone requires archival; archival blocks starting the next milestone. The PM layer becomes a prerequisite tree that must be maintained before any actual work happens.

GSD's own autonomous.md shows the right instinct: `workflow.skip_discuss=true` eliminates ceremony for infrastructure phases, and the system writes minimal CONTEXT.md automatically. The PM layer must have equivalent escape hatches.

**Why it happens:**
PM systems are designed by people who want to track everything. Tracking requires state. State requires transitions. Transitions require validation. The system accumulates validation rules until it enforces a workflow that is more rigid than the work itself.

**Consequences:**
- Developer bypasses the PM layer and works directly (defeating the purpose)
- PM state drifts from reality because keeping it current costs more than the value it provides
- The pipeline commands (`/seraphim:run`, `/seraphim:pause`, `/seraphim:resume`) accumulate PM pre-checks that slow them down

**Prevention:**
- Every PM operation must have a `--quick` or `--no-pm` flag that bypasses PM tracking and goes straight to the pipeline
- PM state is optional metadata, not a gate. The pipeline runs whether or not PM state is current
- Never make pipeline commands depend on PM state being valid. PM reads from pipeline; pipeline does not read from PM
- Design the PM layer as a read path (observe what happened) not a write path (control what can happen)

**Detection:**
- A pipeline command gains a PM pre-check
- A new required field appears in any PM data structure
- Completing a feature requires more than two PM actions (mark-complete is one action; anything more is too much)

**Phase to address:** Feature queue and pipeline integration phase. The boundary must be explicit: PM layer observes pipeline execution; it never blocks it.

---

### Pitfall 3: File-Based State Inconsistency Across Projects

**What goes wrong:**
Seraphim's current state lives in `.seraphim/phases/<phase-id>/state.json` per project. The PM layer adds cross-project state: a feature queue that spans projects, a global roadmap, milestone tracking across the seraphim project and any projects it manages. When the PM state lives in files and multiple projects reference the same logical entities (milestones, features), you get split-brain: project A's state.json says feature X is "in-progress"; the global feature queue says it is "queued". There is no transaction; there is no lock; there is no single source of truth.

This is the fundamental ACID problem. File-based state lacks atomicity: if a feature moves from "queued" to "in-progress", two writes must succeed (queue file, project state). If the second write fails mid-session, the state is permanently inconsistent. There is no rollback.

**Why it happens in this specific context:**
The existing phase-state.js writes individual JSON files per phase per project. This works fine for isolated per-project state. It breaks when the PM layer needs to read across multiple projects simultaneously (cross-project dashboard) or write to shared state (a feature moves from the global queue into a specific project's pipeline).

**Consequences:**
- Dashboard shows stale or conflicting data (project says phase 3 is complete; PM queue says phase 3 is pending)
- Duplicate feature entries (same feature queued twice because the completed state was written to the project but not to the queue)
- Loss of PM state on crash mid-write (write fails after clearing old state but before writing new state)

**Prevention:**
- Treat the project's `.seraphim/` directory as the single source of truth. The global PM layer reads FROM projects; it never writes TO them directly
- Global state (feature queue, roadmap index, milestone registry) lives in one canonical location (`~/.claude/plugins/seraphim/pm/` or equivalent) and is append-only JSONL, not overwritten JSON
- All PM state mutations use write-then-rename (atomic file swap): write to `.tmp` file first, then `mv` to target — this is atomic on Linux
- Never read PM state that is older than the project's last pipeline run timestamp; re-scan project state on access instead of caching

**Detection:**
- Any PM write touches more than one file
- Any PM read joins data from two different file locations without a freshness check
- The PM layer has a concept of "syncing" between the global state and project state (sync implies they can diverge)

**Phase to address:** Data model phase (whichever phase defines how PM state is stored). The storage strategy must be decided before any features are built on top of it.

---

### Pitfall 4: Session Continuity — PM State Orphaned When Pipeline Pauses Mid-Feature

**What goes wrong:**
The existing pause/resume commands in `/seraphim:pause` and `/seraphim:resume` persist pipeline phase state (which of the six phases was active). The PM layer adds a second layer of state: which feature is active, what milestone it belongs to, what its progress is. When a session ends mid-feature, two state files must both be current: the pipeline's `state.json` (which pipeline phase) and the PM layer's feature record (which feature, what progress).

If the PM state is not written at pause time, resume restores the pipeline phase but not the PM context. The developer sees the pipeline resume at "forge" but the feature queue still shows the feature as "queued" — the PM layer lost track of the work even though the pipeline didn't.

The existing `/seraphim:pause` only writes `paused: true`, `paused_at`, and `current_pipeline_phase` to state.json. There is no PM state write. The PM layer must hook into this or define its own pause/resume contract.

**Why it happens:**
Pause/resume is implemented once for the pipeline and considered complete. The PM layer is added later by a different phase. The PM layer assumes it can reconstruct its state from what the pipeline already persists. It cannot — the pipeline state is phase-centric, not feature-centric.

**Consequences:**
- Feature queue shows features as "queued" even after work has started
- Progress tracking resets when sessions end (no in-session progress is preserved)
- Resume from a paused session requires manually re-linking the pipeline phase to the PM feature

**Prevention:**
- Extend `/seraphim:pause` to accept an optional PM context block: active feature ID, milestone ID, progress snapshot. Write this alongside the pipeline state in the same file (same atomic write)
- `/seraphim:resume` reads the PM context block and restores both pipeline and PM state in one step
- If PM context is absent from a state file (backward compatibility with pre-PM-layer sessions), resume works normally; PM layer starts fresh for that session
- Make this an additive change: do not modify the existing state.json schema; extend it with a `pm` key

**Detection:**
- A resume session cannot tell what feature was being worked on before the pause
- The feature queue requires manual status updates after every session end
- PM state and pipeline state are written in separate operations (two writes, not one atomic extend)

**Phase to address:** The pause/resume integration must be in scope for whichever phase builds the feature queue. It cannot be deferred — once the feature queue is built without pause/resume integration, retrofitting it requires touching two separate command files.

---

## Moderate Pitfalls

---

### Pitfall 5: Roadmap Becoming a Second REQUIREMENTS.md

**What goes wrong:**
GSD already has ROADMAP.md, STATE.md, PROJECT.md, and REQUIREMENTS.md at `.planning/`. The Seraphim PM layer adds its own roadmap concept. If the PM roadmap is not clearly differentiated from the GSD planning infrastructure, developers end up maintaining two roadmaps: one in `.planning/ROADMAP.md` (GSD format) and one in the PM layer. They drift. The GSD roadmap reflects execution history; the PM roadmap reflects intent. Neither is complete.

**Prevention:**
- Define the PM roadmap as the source of intent (what to build, in what order, for which milestone). GSD's ROADMAP.md is the execution log (what was built, in what phases, with what outcomes).
- Consider whether the PM layer should consume GSD's ROADMAP.md rather than replace it. If Seraphim is a plugin running inside a project that already uses GSD, the PM layer should read GSD artifacts, not duplicate them.
- If the project IS the Seraphim plugin itself (self-hosted PM), the PM layer reads its own `.planning/ROADMAP.md` — no duplication needed because there is only one project.

**Phase to address:** Roadmap creation phase — the distinction must be in the data model before any UI shows both.

---

### Pitfall 6: Progress Tracking Without a Definition of "Done"

**What goes wrong:**
Progress tracking requires knowing when a feature is complete. The six-phase pipeline has a clear definition of done: Crucible passes verification. But PM-level "done" is ambiguous: is a feature done when it exits the pipeline? When its tests pass? When it is deployed? When the user validates it? If this is not defined upfront, progress tracking shows features as perpetually "in progress" because no automated signal closes them, or it shows them as done too early (pipeline exits but functionality is incomplete).

**Prevention:**
- Define the PM done state explicitly and tie it to an automated signal from the pipeline (the Crucible phase verification PASSED status is the natural trigger)
- Make the done signal configurable per feature type: code features close on Crucible pass; research features close on a human-acceptance flag; mixed features require both
- Build a "stale in-progress" detector: any feature that has been "in-progress" for more than N days without a pipeline event emits a warning

**Phase to address:** Feature queue design phase.

---

### Pitfall 7: Human Task Management Mixing Concern Levels

**What goes wrong:**
The v3.1 target features include "human task management — research, decisions, skills development." This is a separate concern from the AI execution pipeline. Human tasks (read a paper, make a decision, learn a skill) have different lifecycles, different urgency, and different completion signals than pipeline features. If they live in the same queue as pipeline features, the queue becomes noisy and humans deprioritize it because the signal-to-noise ratio drops.

**Prevention:**
- Separate queues: feature queue (AI execution pipeline) and human task queue (research, decisions, skills). Display them separately on the dashboard. Never mix them.
- Human tasks can reference pipeline features ("this decision unblocks feature X") but they are not pipeline items
- Human task completion is always manual — never try to auto-close human tasks from pipeline events

**Phase to address:** Human task management phase — define the separation before the UI shows both.

---

### Pitfall 8: Cross-Project Dashboard Latency From File Scanning

**What goes wrong:**
The existing multi-project dashboard (v3.0) uses `multi-project-scanner.js` to scan all projects for token logs. This works because token logs are small, append-only, and scanning is a read-only operation on static files. A PM layer adds mutable state files (feature queues, roadmaps, milestone status) that must be current when the dashboard renders. If the dashboard scans all projects every time it renders, the latency grows linearly with the number of projects. If it caches PM state, the cache drifts from reality.

**Prevention:**
- PM state is event-driven, not scan-driven: when a feature moves states in a project, that project writes a PM event to a shared event log (`~/.claude/plugins/seraphim/pm/events.jsonl`). The dashboard reads from the event log, not from individual project scans.
- The dashboard computes current PM state by replaying the event log (CQRS pattern — event sourcing for the PM layer). This is consistent, cache-friendly, and supports audit trails.
- Project scans are reserved for initial bootstrapping and manual refresh, not regular dashboard renders.

**Phase to address:** Dashboard extension phase for PM metrics.

---

## Minor Pitfalls

---

### Pitfall 9: Milestone Archival Blocking Forward Progress

**What goes wrong:**
If closing a milestone is required before starting the next one, any incomplete milestone item blocks all new work. In practice, milestones always have incomplete items (deferred features, known gaps, technical debt). If archival requires 100% completion, it never happens. If archival is optional, it gets skipped and the milestone list grows unbounded.

**Prevention:**
- Archival marks a milestone as "shipped" at whatever completion percentage it is at. Incomplete items carry forward to the next milestone automatically.
- The archive command is `/seraphim:archive-milestone <version>` — it runs without blocking, moves the milestone to an archive directory, and generates a carryforward list for the next milestone.
- Never gate new milestone creation on archival of the prior milestone.

---

### Pitfall 10: Version Naming Collision Between GSD Milestones and Seraphim PM Versions

**What goes wrong:**
GSD uses `milestone_version` (e.g., `v3.1`) for its own tracking. The Seraphim PM layer also tracks versions. If both systems use the same version string for the same project, they will conflict when GSD tooling reads the version number and finds PM-layer metadata it doesn't understand.

**Prevention:**
- Seraphim PM versions use a different namespace prefix: `pm-v1.0`, `pm-v2.0`. GSD milestone versions remain `v3.1`, `v3.2`, etc.
- Or, simply: the Seraphim PM layer does not use version strings as primary keys. It uses UUIDs or human-readable slugs for milestones. Version strings are display metadata only.

---

## Phase-Specific Warnings

| Phase Topic | Likely Pitfall | Mitigation |
|-------------|----------------|------------|
| Roadmap creation and milestone planning | Second system effect: roadmap spec grows to include sprint planning, velocity, and dependency graphs | Define anti-features before the Discover phase; cap the feature list |
| Feature queue design | Ceremony creep: feature queue blocks pipeline invocation | PM layer is read-path only; pipeline never waits on PM state |
| Feature queue design | Missing definition of "done" | Tie closure signal to Crucible verification pass event |
| Human task management | Mixing human tasks with pipeline features in one queue | Separate queues; separate dashboard panels |
| Progress tracking per feature/phase | File consistency: two files must update atomically | Use append-only event log + atomic file swap; never dual-write |
| Session pause/resume with PM context | PM state orphaned on pause | Extend pause command's state.json write to include PM context block |
| Dashboard extension | Scan latency grows with project count | Event-driven PM state; dashboard reads event log, not project files |
| Milestone archival | Archival gates new milestone creation | Archive at any completion %; carry forward incomplete items |
| Cross-project PM overview | PM roadmap duplicates GSD ROADMAP.md | Define distinct roles: PM = intent, GSD = execution log |

---

## Sources

- Fred Brooks, "The Mythical Man-Month" — second system effect (canonical; widely verified)
- Wikipedia: [Second-system effect](https://en.wikipedia.org/wiki/Second-system_effect) — definition and prevention strategies
- Oracle Developers / Richmond Alake, Medium (Feb 2026): [Comparing File Systems and Databases for Effective AI Agent Memory Management](https://medium.com/oracledevs/comparing-file-systems-and-databases-for-effective-ai-agent-memory-management-5322ac45f3b6) — file-based vs database pitfalls for AI state (HIGH confidence, recent and domain-specific)
- GSD `autonomous.md` at `~/.claude/get-shit-done/workflows/autonomous.md` — observed directly; workflow.skip_discuss and escape hatches confirm the ceremony-creep risk is real and GSD has already solved parts of it
- Seraphim `pause.md` / `resume.md` at `~/.claude/plugins/seraphim/commands/` — observed directly; PM state gap is confirmed by reading what pause currently persists
- Complex.so: [Why PM tools fail small teams](https://complex.so/insights/why-most-project-management-tools-fail-small-teams-(and-what-to-use-instead)) — ceremony creep and overhead evidence (MEDIUM confidence, single source)
- Lullabot: [Building an AI-Powered Project Management System](https://www.lullabot.com/articles/building-ai-powered-project-management-system) — context loss between sessions pattern (MEDIUM confidence)
