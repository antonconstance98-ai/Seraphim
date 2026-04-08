# Phase 9: Human-AI Cognitive Division - Context

**Gathered:** 2026-04-08
**Status:** Ready for planning

<domain>
## Phase Boundary

Research where human cognition vs AI execution is optimally placed within Seraphim's six-phase pipeline. Produce concrete, code-level changes to the pipeline — not a standalone document. Findings are embedded directly into pipeline behavior: checkpoint defaults, phase prompts, timing suggestions, and project-specific skill recommendations on the dashboard.

</domain>

<decisions>
## Implementation Decisions

### Human Checkpoints in Pipeline
- **D-01:** Three mandatory human gates in every pipeline run: (1) before Envision — human reviews discovery and seeds creative direction, (2) after Judge, before Architect — human makes final call on which approach to pursue, (3) after Crucible — human reviews finished product before accepting.
- **D-02:** Gates are mandatory by default. When user runs in full-auto mode (e.g., `--auto`), the system still asks which gates the human wants to be present for and lets them select/deselect individual gates. The human always has the choice — full-auto doesn't mean no human option.
- **D-03:** Pipeline pauses at each active gate and waits for human input. No timeout — human decides when to engage.

### Research Output Format
- **D-04:** Research findings are embedded directly into the pipeline — no separate framework document. The code IS the research output. Findings become: checkpoint defaults, phase prompt adjustments, model selection rationale comments, and config options.
- **D-05:** Evidence citations from the divergent/convergent research are preserved as comments in the code, not as a separate doc. Downstream maintainers can trace why a checkpoint exists.

### Workflow Scheduling (Chronobiology)
- **D-06:** Seraphim includes optional time-of-day awareness based on chronobiology research. Advisory only — suggests optimal timing for human-gate phases (e.g., "Morning is good for Envision review — divergent thinking peaks during groggier periods").
- **D-07:** Timing suggestions are toggle-able via config (`timing_hints: true/false`). Off by default — user opts in.
- **D-08:** Based on research: Wieth & Zacks (2011) — insight problems solved better during non-optimal alertness; Colzato et al. (2012) — Open Monitoring meditation promotes divergent thinking; Oppezzo & Schwartz (2014) — walking produces 60% creative boost.

### Project-Specific Skill Recommendations
- **D-09:** Dashboard panel shows skill development recommendations specific to the current project's domain. Not generic cognitive advice — project-aware suggestions.
- **D-10:** System analyzes project content (phase outputs, blueprint tasks, domain signals) to identify knowledge areas where human expertise would improve outcomes. E.g., "This project involves sales/persuasion — human expertise in persuasion frameworks would strengthen Envision and Judge phases."
- **D-11:** Skill recommendations surface on the Phase 7 web dashboard, not in terminal output.

### Claude's Discretion
- Specific prompt modifications per phase based on cognitive division findings
- How timing suggestions are calculated (chronotype detection vs simple time-of-day)
- Skill recommendation algorithm (keyword matching, semantic analysis, or LLM-based)
- How evidence citations are formatted in code comments

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Divergent/Convergent Research
- `~/projects/thought-orphanage/seeds/[exploring] divergent-convergent-thought/SEED.md` — Core thesis: where human leverage is protected in an AI world
- `~/projects/thought-orphanage/seeds/[exploring] divergent-convergent-thought/RESEARCH.md` — Full evidence base: AI vs human creativity, cognitive advantages taxonomy, chronobiology, enhancement practices, information diet research

### Design Spec
- `docs/specs/2026-04-04-seraphim-v3-design.md` — Six-phase pipeline definitions, model assignments per profile

### Prior Phase Context
- `.planning/phases/03-six-phase-pipeline-and-profile-management/03-CONTEXT.md` — Pipeline structure decisions (D-06: terminal output style, D-07: six-wing visual identity)
- `.planning/phases/06-adaptive-intelligence/06-CONTEXT.md` — Analysis from first run, dashboard integration decisions

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- No plugin source code exists yet — Phase 3 (pipeline) builds the code this phase modifies
- `~/projects/thought-orphanage/` — Thought-orphanage project with braindump workflow and seed management patterns

### Established Patterns
- Pipeline phases defined as markdown command files in plugin
- Phase-state.js persists loop counters and completion flags
- Config.js reads/writes .seraphim/config.json with validation

### Integration Points
- Human gates integrate into the `/seraphim:run` orchestrator (Phase 3, plan 03-06)
- Timing config goes into `.seraphim/config.json` via config.js
- Skill recommendations display on Phase 7 web dashboard

</code_context>

<specifics>
## Specific Ideas

- User explicitly wants project-specific skills, not generic cognitive advice. Example: if a project needs sales/persuasion knowledge, surface that — don't just say "practice divergent thinking."
- Human gates should feel like a natural pause, not an interruption. The system presents what it found and asks for direction.
- Full-auto mode still respects human agency — it asks which gates to skip, not assumes all are skipped.

</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope

</deferred>

---

*Phase: 09-human-ai-cognitive-division*
*Context gathered: 2026-04-08*
