# Domain Pitfalls: Idea-to-Shipped Workflow Features on an Existing PM Plugin

**Domain:** Adding idea capture, research system, requirements definition, phased roadmaps, planning, verification, enriched human tasks, and progress visualization to Seraphim — which already has a working PM layer (v3.1) and six-phase execution pipeline (v3.0)
**Context:** v3.2 subsequent milestone — all three prior layers (pipeline, PM, dashboard) are shipped and must not be broken
**Researched:** 2026-04-09
**Confidence:** HIGH for duplication and schema drift pitfalls (directly observable in existing codebase); HIGH for markdown command runtime interpretation pitfalls (observed from plugin architecture); MEDIUM for workflow ceremony and verification anti-pattern pitfalls (pattern-based, confirmed by multiple community sources)

---

## Critical Pitfalls

---

### Pitfall 1: Duplicating Existing PM Primitives Instead of Extending Them

**What goes wrong:**
v3.2 introduces "phased roadmaps with waves, dependencies, and success criteria." v3.1 already ships `roadmap.json` with milestones, feature queues, WIP limits, and dependency guards. If the v3.2 roadmap concept is designed without reading the v3.1 schema first, it creates a parallel roadmap representation — two files, two schemas, two update paths, neither complete without the other.

The same trap applies to requirements. v3.1 has a feature queue (`add-feature.md`) where features have IDs, slugs, status, and dependencies. v3.2 wants REQ-IDs and v1/future/out-of-scope scoping. If REQ-IDs are a new concept instead of a field added to the existing feature record, you get feature queue entries that don't match requirement entries — manual cross-referencing for every query.

**Why it happens:**
New milestone, new researcher subagent, new planning session. The prompt says "add requirements definition." The agent designs a requirements system from scratch because it hasn't been explicitly told the feature queue already exists. This is a coordination failure between planning phases, not a technical failure.

**How to avoid:**
Before designing any v3.2 data structure, read `.seraphim/roadmap.json` schema and `~/.claude/plugins/seraphim/pm/` directory. Every v3.2 concept must map to an existing field (extend it) or a genuinely new concept (add it once). The question to ask: "Does this already exist under a different name?"

Specifically:
- "Wave" = a group of milestones. Add `wave_id` to the existing milestone object; do not create a waves.json.
- "REQ-ID" = a field on an existing feature record. Add `req_id` to the feature queue entry; do not create requirements.json.
- "Success criteria" = a field on an existing milestone object. Extend the milestone schema; do not create a separate criteria store.

**Warning signs:**
- A planning phase proposes a new JSON file when an existing file already stores the parent concept
- The phrase "we'll need a separate store for..." appears in any plan
- Two data files reference each other by ID but neither is declared the single source of truth

**Phase to address:** Data model / roadmap schema phase (first phase of v3.2). Lock the schema extensions before building any commands on top of them.

---

### Pitfall 2: Research System Bypassing the Human-Interrogation Step

**What goes wrong:**
The v3.2 research system includes "human interrogation before AI research." This is the right instinct — AI research on a poorly-specified question produces authoritative-sounding noise. But the implementation commonly skips the interrogation step under time pressure: the command fires immediately, the AI generates a research brief, and the human never redirects it. The interrogation becomes optional, then forgotten, then removed.

The same failure happened in GSD's `/gsd:new-project` flow. The original design required a human scoping session before research spawned. In practice, research agents ran in parallel with scoping, and the human never saw the questions before the brief was done.

**Why it happens:**
Interrogation requires a synchronous pause — the command must stop, ask questions, and wait. In an AI-orchestrated system where agents run autonomously, synchronous pauses feel like bugs. The path of least resistance is async: ask questions in the output, but don't block on answers. Then the questions are never answered.

**How to avoid:**
The research command must be two distinct slash commands, not one with an optional flag:
1. `/seraphim:research-scope {topic}` — generates questions only, writes them to a `research-brief.md` stub, exits. Human edits the stub.
2. `/seraphim:research-run {brief}` — reads the answered stub and executes. Refuses to run without answered questions in the brief.

Never merge these into a single command with `--skip-interrogation`. The separation is the feature.

**Warning signs:**
- The research command has a `--auto` or `--quick` flag that skips questions
- Questions are printed to stdout but the command continues running before answers are provided
- The research brief template has sections marked "optional"

**Phase to address:** Research system design phase. The two-command architecture must be specified in the plan before any implementation begins.

---

### Pitfall 3: Neon Schema Divergence Between v3.1 PM Tables and v3.2 Workflow Tables

**What goes wrong:**
v3.1 shipped Neon sync with known issues: the DDL was never manually applied, the schema uses `project` while old tables use `project_name`, and `cost_usd` is a stub. v3.2 adds new workflow concepts (research briefs, requirements, roadmap waves) that need Neon columns. If v3.2 adds columns to a schema that is already inconsistent, migration becomes a multi-step operation where each step has dependencies on unresolved v3.1 debt.

Concretely: if v3.2 adds a `requirements` table that joins to `features` on `feature_id`, but the v3.1 `features` table never had its DDL applied, the join silently returns empty results rather than erroring. The dashboard shows no requirements data. Debugging requires tracing back through three schema versions.

**Why it happens:**
v3.2 planning assumes v3.1 shipped cleanly. It didn't — the tech debt list explicitly flags "Neon DDL pending manual application" and "feature_id not passed to decisions-logger." New feature design proceeds without checking whether the foundation it builds on exists.

**How to avoid:**
Phase 1 of v3.2 must be a debt-clearance gate for the specific v3.1 items that v3.2 depends on:
- Apply the pending Neon DDL and verify with a `SELECT 1` query
- Confirm `feature_id` is flowing through the decisions logger
- Reconcile `project` vs `project_name` schema inconsistency before adding any new columns

No v3.2 Neon work starts until these pass. This is not optional cleanup — it is a prerequisite.

**Warning signs:**
- A v3.2 plan references a Neon table that was planned in v3.1 but not confirmed as created
- A new column is added to a table without checking whether that table has its base schema applied
- Dashboard queries return empty results that are attributed to "no data yet" rather than schema absence

**Phase to address:** Phase 1 of v3.2 — tech debt clearance before any new schema work.

---

### Pitfall 4: Markdown Commands Accumulating Logic They Cannot Test

**What goes wrong:**
Seraphim's plugin architecture uses markdown commands that Claude interprets at runtime. This is powerful for natural-language workflow orchestration. It is dangerous when commands accumulate conditional logic, state checks, and error paths written in prose. A markdown command that does "if the research brief has answered questions, call research-run; otherwise, remind the user to answer questions; if the brief is missing, create it first; if the topic is ambiguous, ask for clarification" has four branches that cannot be unit tested, cannot be linted, and cannot be debugged with a stack trace.

v3.2 is the highest-complexity milestone yet — seed capture, research, requirements, roadmap, planning, verification, and enriched tasks are all markdown commands. Each one will drift toward prose-encoded business logic as edge cases accumulate.

**Why it happens:**
It is faster to add "and if X happens, do Y" to a markdown command than to write a Node.js helper and call it from the command. The first five edge cases get handled this way. By the tenth edge case, the command is unreadable and untestable.

**How to avoid:**
Establish a rule at the start of v3.2: markdown commands contain workflow narrative (what to do, in what order, for the human). All conditional logic and state manipulation lives in Node.js helper scripts in `~/.claude/plugins/seraphim/lib/`. The command calls the helper; the helper does the work.

Example: the research scope command's question generator is a Node.js function that accepts a topic and returns structured questions. The markdown command calls that function and presents the output. The function is testable; the prose is not.

**Warning signs:**
- A markdown command contains the word "if" more than three times
- A markdown command has more than one `---` separator (indicating multiple branches)
- Two commands share similar logic written in prose rather than calling a shared helper

**Phase to address:** Architecture definition phase — establish the markdown-vs-Node.js boundary rule before any commands are written.

---

### Pitfall 5: Verification Becoming a Checkbox Instead of a Gate

**What goes wrong:**
v3.2 includes "goal-backward verification and UAT." The correct implementation: verification asks "does the output satisfy the original goal?" and blocks progress if the answer is no. The common failure: verification becomes a checklist that the AI marks complete autonomously, with no human review required for any item.

When an AI verifies its own output against criteria it helped write, the verification rate approaches 100% — not because the output is correct but because the AI pattern-matches criteria to output without genuine adversarial evaluation. GSD's Crucible phase has this problem in practice: the static analysis passes because the review criteria were generated from the same reasoning context as the code.

**Why it happens:**
Autonomous verification is faster. The human is the bottleneck. The system is designed to minimize human interrupts. Verification gets added to the list of things the AI handles, defeating its purpose.

**How to avoid:**
Verification has two required components that cannot both be AI-owned:
1. Criteria generation — AI-assisted (it knows what was built); human-reviewed (the human confirms the criteria match the original intent)
2. Criteria evaluation — AI-assisted for mechanical checks (tests pass, schema matches); human-required for intent checks (does this actually solve the problem?)

The `/seraphim:verify` command must output a verification report that includes at least one item flagged `REQUIRES_HUMAN_JUDGMENT`. If it has zero such items, the verification is incomplete. Build this check into the command.

**Warning signs:**
- A verification run completes without any human task being generated
- The verification output consists entirely of checkmarks with no open questions
- The command has a `--auto-approve` flag

**Phase to address:** Verification system design phase. The human-judgment requirement must be in the spec before implementation.

---

## Technical Debt Patterns

Shortcuts that seem reasonable but create long-term problems.

| Shortcut | Immediate Benefit | Long-term Cost | When Acceptable |
|----------|-------------------|----------------|-----------------|
| Adding fields to roadmap.json without updating the TypeScript/JSDoc type definition | Faster development | Silent type errors, dashboard renders wrong fields | Never — the schema is the contract |
| Writing research output as free-form markdown instead of structured JSON | Easier to read immediately | Cannot be queried, diffed, or fed to subsequent pipeline phases automatically | Only for human-only outputs (never for machine-consumed artifacts) |
| Hardcoding v3.2 workflow names (wave, req_id) in dashboard queries instead of reading from schema | Faster to build | Any schema rename breaks the dashboard silently | Never — read field names from a config constant |
| Using TODO comments in markdown commands for edge cases | Defers decisions | Edge cases remain unhandled indefinitely; markdown TODOs are never surfaced by linters | Only if accompanied by a human task in the inbox |
| Implementing "enriched human tasks" as a new task format without migrating existing human tasks | Avoids migration risk | Two task formats in the inbox; dashboard and filters must handle both | Never — migrate at introduction or use a format version field |

---

## Integration Gotchas

Common mistakes when connecting to the existing v3.1 system.

| Integration | Common Mistake | Correct Approach |
|-------------|----------------|------------------|
| Roadmap waves → existing milestone schema | Create a new `waves.json` alongside `roadmap.json` | Add `wave_id` and `wave_name` fields to the existing milestone object in `roadmap.json` |
| Requirements → existing feature queue | Create `requirements.json` with its own ID sequence | Add `req_id`, `scope` (v1/future/out-of-scope) fields to the existing feature record |
| Research briefs → dashboard | Build a separate research panel from scratch | Extend the existing human tasks panel; research tasks are a task type, not a separate panel |
| Verification results → Neon | Create a new `verifications` table | Add `verification_status` and `verified_at` columns to the existing features table |
| Progress bars → existing pipeline phases | Re-implement phase tracking | Read `phase-state.js` output; the phase completion data already exists |
| Seed capture → feature queue | Create a `seeds.json` staging area | Seeds are features with `status: seed`; they live in the feature queue with a seed status |

---

## Performance Traps

Patterns that work now but degrade with scale.

| Trap | Symptoms | Prevention | When It Breaks |
|------|----------|------------|----------------|
| Re-parsing full `roadmap.json` on every dashboard render for wave progress | Dashboard sluggish as roadmap grows | Cache parsed roadmap in memory; invalidate only on write | Above ~50 features |
| Running research interrogation questions through a full Opus call | 30-60 second latency for a question list | Use Haiku or Sonnet for question generation; Opus for synthesis only | Every research invocation |
| Storing research briefs as embedded markdown in `roadmap.json` | JSON file grows large; parsing slows | Store briefs as separate files (`research/{req-id}.md`); store only the file path in `roadmap.json` | Above ~10 research items |
| Blocking the dashboard render on Neon queries | Dashboard timeout if Neon is unreachable | Render from local files first; Neon data is additive enhancement, never blocking | Neon connection issues |

---

## UX Pitfalls

Common experience mistakes specific to idea-to-shipped workflow commands.

| Pitfall | User Impact | Better Approach |
|---------|-------------|-----------------|
| Seed capture requires structured input (category, priority, etc.) at capture time | User loses the idea rather than capturing it imperfectly | Seed capture is a braindump — one command, raw text input, zero required fields. Enrichment is a separate step. |
| Research brief questions are generic (not tailored to the specific topic) | Human answers questions that don't apply; useful questions are missing | Question generator reads the existing roadmap context before generating questions; questions reference specific features and milestones |
| Verification report shows only pass/fail with no evidence | Human cannot evaluate whether the AI's judgment was correct | Every verification item includes the evidence that was checked (file path, line, output excerpt) |
| Progress bars show pipeline phase completion but not milestone completion | Human cannot answer "how far through v3.2 are we?" | Two progress metrics: pipeline progress (phases complete for current feature) and milestone progress (features complete for current milestone) |
| Discuss phase is optional and consistently skipped | Architecture decisions are never locked; plans drift from intent | Discuss phase generates a decision record that subsequent planning phases read. If discuss was skipped, planning phase warns that decisions are unverified. |

---

## "Looks Done But Isn't" Checklist

Things that appear complete but are missing critical pieces.

- [ ] **Seed capture command:** Often missing de-duplication check — verify that capturing the same idea twice creates one record, not two
- [ ] **Research system:** Often missing the refusal-to-run check — verify that `research-run` refuses when brief has unanswered questions
- [ ] **Requirements command:** Often missing the scope field on existing features — verify that features created before v3.2 display gracefully when `req_id` is absent
- [ ] **Phased roadmap:** Often missing wave ordering enforcement — verify that wave 2 features cannot be started before wave 1 features are complete (or that the system warns)
- [ ] **Verification:** Often missing the human-judgment item — verify that every verification report contains at least one item requiring human sign-off
- [ ] **Progress bars:** Often missing the "no data" state — verify bars render correctly when a milestone has zero features (empty state, not NaN%)
- [ ] **Enriched human tasks:** Often missing migration of existing task format — verify that v3.1 human tasks (decision/research/review/validation/skills) display correctly in the v3.2 inbox
- [ ] **Dashboard control center:** Often missing the error boundary — verify that a Neon failure does not crash the entire dashboard page (v3.1 tech debt item, must be closed in v3.2)

---

## Recovery Strategies

| Pitfall | Recovery Cost | Recovery Steps |
|---------|---------------|----------------|
| Duplicate PM primitives discovered mid-milestone | HIGH | Stop. Merge schemas before writing any more commands. Every command written on the wrong schema is a rewrite. |
| Research system shipped without interrogation gate | MEDIUM | Add `research-run` refusal check as a hotfix. Re-run any research briefs that were generated without human input. |
| Neon schema divergence discovered after v3.2 data is written | HIGH | Write a migration script. Apply to staging first. Verify row counts before and after. This is the most expensive recovery in this list. |
| Markdown commands with untestable logic | MEDIUM | Extract logic to Node.js helpers incrementally. Do not rewrite the command — just move the conditional into a helper and call it. |
| Verification passes with zero human-judgment items | LOW | Add the check and re-run. The cost is one additional human review per verification report. |

---

## Pitfall-to-Phase Mapping

| Pitfall | Prevention Phase | Verification |
|---------|------------------|--------------|
| Duplicate PM primitives | Phase 1 — schema extension audit | Read roadmap.json after schema changes; confirm zero new top-level files were added |
| Research without interrogation gate | Research system design phase | Attempt to run `research-run` without answering brief questions; confirm refusal |
| Neon schema divergence | Phase 1 — v3.1 debt clearance | Run all v3.1 Neon queries; confirm non-empty results before writing v3.2 columns |
| Markdown commands accumulating logic | Architecture definition (pre-build) | Count "if" occurrences in each command file; any above 3 requires a Node.js helper |
| Verification as checkbox | Verification system design phase | Run verify on a deliberately broken output; confirm at least one item is flagged for human judgment |
| Tech debt blocking new schema work | Phase 1 — explicit debt gate | All 8 v3.1 tech debt items reviewed; items that are v3.2 prerequisites closed before Phase 2 starts |

---

## Sources

- Seraphim `PROJECT.md` (v3.1 tech debt list, 8 items) — read directly 2026-04-09
- Seraphim `STATE.md` (v3.1 decisions log, Neon DDL pending manual application) — read directly 2026-04-09
- Seraphim v3.1 PITFALLS.md (second system effect, ceremony creep, file consistency, session continuity) — prior research, informs what pitfalls are already addressed at the PM layer level
- GSD `autonomous.md` — interrogation bypass pattern observed directly in gsd:new-project flow
- Seraphim plugin architecture (markdown commands at `~/.claude/plugins/seraphim/commands/`) — untestable logic risk is directly observable from the command format
- v3.1 MILESTONE-AUDIT (deleted from tree per git status, but referenced tech debt in PROJECT.md) — 8 tech debt items confirmed from PROJECT.md

---
*Pitfalls research for: v3.2 idea-to-shipped workflow features on existing Seraphim PM plugin*
*Researched: 2026-04-09*
