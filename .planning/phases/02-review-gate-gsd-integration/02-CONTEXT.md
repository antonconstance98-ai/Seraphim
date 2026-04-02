# Phase 2: Review Gate & GSD Integration - Context

**Gathered:** 2026-04-02
**Status:** Ready for planning

<domain>
## Phase Boundary

Wire the Codex review gate into Claude Code's Stop hook and integrate Codex checkpoints into GSD workflows. This phase delivers: Stop hook review gate with ALLOW/BLOCK protocol, bidirectional cross-model review, GSD plan-phase review checkpoint, GSD wave-boundary validation, and global routing hook for non-GSD workflows. No plan review loops (Phase 3), no Superpowers integration (Phase 3), no cost reporting (Phase 4).

</domain>

<decisions>
## Implementation Decisions

### Review Gate Trigger
- **D-01:** Stop hook triggers Codex review **only when code changes are present** — Write/Edit/Bash that modified files. Chat-only responses skip review. Avoids unnecessary Codex calls and latency
- **D-02:** Review gate is **not bypassable** — if enabled for a project, it always runs. Consistency over speed. Quality enforced on every code change
- **D-03:** When Codex finds an issue, it **blocks and Opus fixes** the issue before the user sees anything. Quality enforced automatically via the ALLOW/BLOCK protocol

### Review Feedback Visibility
- **D-04:** User sees a **one-line summary**: "Codex reviewed: PASS" or "Codex reviewed: fixed [issue]". Minimal noise, clear signal
- **D-05:** When Codex blocks and Opus fixes, user sees a **brief note**: "Codex caught: [issue]. Fixed before delivery." Transparency without verbosity
- **D-06:** Review events are **logged to the existing token log** (`.planning/token-log.jsonl`). Phase 4 cost reports can show review activity alongside token usage. No separate review log file

### GSD Checkpoint Behavior
- **D-07:** Plan-phase finalization is **blocked until Codex reviews**. Codex feedback gets incorporated into the plan file before execution starts
- **D-08:** Wave-boundary validation is **non-blocking** — Codex validates in the background while execution continues. Results surface at natural stopping points
- **D-09:** **Critical issues halt the next wave** — if Codex flags a critical issue (broken imports, security flaw) during wave validation, the next wave is blocked until resolved. Non-critical issues remain advisory

### Review Scope
- **D-10:** Codex reviews for **all four categories**: bugs/logic errors, security vulnerabilities, requirements alignment, and style/conventions
- **D-11:** Codex sees the **diff plus relevant surrounding context** — enough to understand impact without reading entire files. Best accuracy-to-cost ratio
- **D-12:** Review depth **varies by task type** — deep review for new features and security-sensitive code, light review for test generation and bulk operations. Optimizes cost vs quality

### Carrying Forward from Phase 1
- D-05 (Phase 1): Codex handles code review as one of its four task types
- D-06 (Phase 1): Tool-call based routing detection
- D-08 (Phase 1): Results are attributed — user knows which model did the work

### Claude's Discretion
- Infinite loop prevention: Claude implements the `stop_hook_active` guard to prevent review-triggers-review loops
- Critical issue classification: Claude defines what counts as "critical" vs "non-critical" for wave-halt decisions
- Review prompt design: Claude crafts the review prompts sent to Codex for each review type (stop hook, plan review, wave validation)
- GSD hook integration points: Claude determines exact hook event names and matcher patterns for GSD workflow integration
- Global routing hook design: Claude extends the Phase 1 routing hook to cover non-GSD workflows

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Research & Benchmarks
- `docs/research/opus-vs-codex-model-comparison.md` — Cross-model review effectiveness data; informs review depth and scope decisions
- `codex-claude-code-power-user-research.md` — Power user patterns including stop-hook review gate patterns

### Project Configuration
- `CLAUDE.md` — Hook event table (PostToolUse, Stop, SubagentStop, TaskCompleted), ALLOW/BLOCK protocol, hook configuration skeleton
- `.planning/REQUIREMENTS.md` — Phase 2 requirements: REVW-01, REVW-02, ROUT-02, GSD-01, GSD-02, GSD-03, GSD-04

### Prior Phase Context
- `.planning/phases/01-foundation/01-CONTEXT.md` — Phase 1 decisions on routing, AGENTS.md, tool-call based detection, attributed results

### Existing Patterns
- `~/.claude/settings.json` — Current hook configuration showing SessionStart, PostToolUse, PreToolUse patterns
- `~/.claude/hooks/gsd-context-monitor.js` — Reference implementation for PostToolUse hook (stdin JSON, timeout guard, additionalContext output)

### GSD Internals (verify before implementing)
- `~/.claude/get-shit-done/bin/lib/state.cjs` — GSD wave state schema (field names flagged as unverified in STATE.md — must confirm before writing wave-boundary hook)
- `~/.claude/get-shit-done/bin/lib/phase.cjs` — Phase execution logic, wave boundary detection

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- **Phase 1 hook infrastructure** (once built): Codex invocation wrapper, token logging, routing logic — Phase 2 extends these
- **6 existing hook scripts** in `~/.claude/hooks/` — all follow the same Node.js stdin/stdout JSON pattern with timeout guards
- **GSD tools CLI** — config management, state tracking, commit helpers available for hook scripts
- **Codex CLI** at `~/.npm-global/bin/codex` v0.118.0 — `codex exec --json` for structured review output

### Established Patterns
- Stop hook can return `continue: false` to block Claude's response
- PostToolUse hooks inject `additionalContext` (max 10,000 chars) into Claude's context
- PreToolUse hooks can return `decision: "block"` with a `reason`
- GSD already has `SubagentStop` and `TaskCompleted` events that could be used for review triggers

### Integration Points
- New Stop hook entry in `~/.claude/settings.json` for review gate
- GSD plan-phase workflow modification for Codex review checkpoint
- GSD execute-phase workflow modification for wave-boundary validation
- `.planning/token-log.jsonl` — append review events alongside token usage

</code_context>

<specifics>
## Specific Ideas

No specific requirements — open to standard approaches based on existing codebase patterns.

</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope.

</deferred>

---

*Phase: 02-review-gate-gsd-integration*
*Context gathered: 2026-04-02*
