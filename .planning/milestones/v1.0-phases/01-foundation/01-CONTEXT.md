# Phase 1: Foundation - Context

**Gathered:** 2026-04-02
**Status:** Ready for planning

<domain>
## Phase Boundary

Build the integration plumbing so Codex can be safely invoked from Claude Code hooks. This phase delivers: AGENTS.md, hook infrastructure for `codex exec --json`, PreToolUse routing, security hardening, and token tracking to JSONL. No review logic, no GSD integration, no plan loops — those come in later phases.

</domain>

<decisions>
## Implementation Decisions

### Codex Briefing (AGENTS.md)
- **D-01:** Guardrails are **moderate** — Codex can make minor judgment calls (naming, small refactors) but must escalate anything structural back to Opus
- **D-02:** AGENTS.md includes a **full project brief** — project description, tech stack, file structure conventions, security rules, and the hard rule that Opus is the sole architect
- **D-03:** AGENTS.md **matches Claude's communication style** — plain English explanations, no jargon, simple solutions. Consistent experience across both models
- **D-04:** AGENTS.md is a **living document** — updated at GSD phase transitions as new conventions emerge. Codex always has current project context

### Routing Boundaries
- **D-05:** Codex handles **all four task types**: clearly-defined implementation, test generation, bulk file operations, and code review / diff analysis
- **D-06:** Routing detection is **tool-call based** — route by what Claude is doing (Write/Edit calls during plan execution go to Codex; architecture, debugging, exploratory work stays with Opus). Uses existing hook matcher pattern
- **D-07:** Routing is **opt-in** — OFF by default, enabled per project via `.claude/settings.json` or a config flag. Safest for first version
- **D-08:** Results are **attributed** — Opus presents the result but notes which model did the work. User always knows when Codex handled something

### Claude's Discretion
- Failure behavior: Claude decides how to handle Codex timeouts/failures (silent fallback vs notification) based on what works best for the experience
- Token log detail level: Claude decides the right granularity for `.planning/token-log.jsonl` based on what Phase 4 cost reporting needs
- Hook script architecture: Claude follows existing `~/.claude/hooks/` patterns (Node.js, stdin JSON, `additionalContext` output)
- Security implementation: Claude handles version verification, env var protection, and SUBPROCESS_ENV_SCRUB setup

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Research & Benchmarks
- `docs/research/opus-vs-codex-model-comparison.md` — Model comparison data from 34 sources; informs all routing decisions and task-type boundaries
- `codex-claude-code-power-user-research.md` — Power user patterns for Codex + Claude Code integration

### Project Configuration
- `CLAUDE.md` — Full technology stack, hook configuration skeleton, installation steps, and "What NOT to Use" constraints
- `.planning/REQUIREMENTS.md` — Phase 1 requirements: FNDTN-01 through FNDTN-05, ROUT-01, ROUT-03, ROUT-04, TRCK-01, TRCK-02

### Existing Patterns
- `~/.claude/settings.json` — Current hook configuration (SessionStart, PostToolUse, PreToolUse) — follow this structure for new hooks
- `~/.claude/hooks/gsd-context-monitor.js` — Reference implementation for PostToolUse hook pattern (stdin JSON, timeout guard, additionalContext output)

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- **6 existing hook scripts** in `~/.claude/hooks/`: gsd-context-monitor.js, gsd-prompt-guard.js, claude-settings-guard.js, gsd-check-update.js, gsd-statusline.js, gsd-workflow-guard.js — all follow the same Node.js stdin/stdout JSON pattern
- **GSD tools CLI** at `~/.claude/get-shit-done/bin/gsd-tools.cjs` — config management, state tracking, commit helpers
- **Codex CLI** at `~/.npm-global/bin/codex` v0.118.0 — confirmed working with `codex exec --json` for structured output

### Established Patterns
- Hooks read JSON from stdin, process, and output JSON with `additionalContext` / `decision` / `reason` fields
- Timeout guards on stdin (10s) prevent hanging when pipe issues occur
- Settings in `~/.claude/settings.json` (user scope) for cross-project hooks; `.claude/settings.json` (project scope) for per-project overrides
- PreToolUse hooks use `matcher` field to target specific tools (e.g., `"Write|Edit"`)

### Integration Points
- New PreToolUse hook entries in `~/.claude/settings.json` for routing
- New PostToolUse hook for token logging after Codex calls
- AGENTS.md at repo root (read by Codex CLI automatically)
- `.planning/token-log.jsonl` for token tracking data

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

*Phase: 01-foundation*
*Context gathered: 2026-04-02*
