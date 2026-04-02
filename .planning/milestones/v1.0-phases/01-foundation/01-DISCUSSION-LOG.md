# Phase 1: Foundation - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md — this log preserves the alternatives considered.

**Date:** 2026-04-02
**Phase:** 01-foundation
**Areas discussed:** Codex briefing (AGENTS.md), Routing boundaries

---

## Codex Briefing (AGENTS.md)

### Q1: How strict should Codex's guardrails be?

| Option | Description | Selected |
|--------|-------------|----------|
| Strict | Codex follows exact instructions only. No creative decisions, no refactoring beyond what's asked, no architectural opinions. Safest for a new integration. | |
| Moderate | Codex can make minor judgment calls (naming, small refactors) but escalates anything structural back to Opus. | ✓ |
| Flexible | Codex can make implementation choices freely as long as it doesn't touch architecture. Most autonomous, but higher risk of unexpected changes. | |

**User's choice:** Moderate
**Notes:** None

### Q2: How much project context should AGENTS.md include?

| Option | Description | Selected |
|--------|-------------|----------|
| Full project brief (Recommended) | Project description, tech stack, file structure conventions, security rules, and the 'Opus is architect' rule. Codex knows the whole picture. | ✓ |
| Minimal — rules only | Just the hard rules (no architecture decisions, security constraints, coding style). Codex gets task-specific context per invocation instead. | |
| You decide | Claude picks the right level of detail based on what works best for hook integration. | |

**User's choice:** Full project brief (Recommended)
**Notes:** None

### Q3: Should AGENTS.md include your CLAUDE.md conventions?

| Option | Description | Selected |
|--------|-------------|----------|
| Yes — match Claude's style | Codex follows the same communication and coding style rules as Claude. Consistent experience across both models. | ✓ |
| No — Codex is code-only | Codex doesn't explain anything to you — it just writes code. Opus handles all communication. Codex output is internal. | |
| You decide | Claude picks based on how Codex output surfaces to you vs stays internal. | |

**User's choice:** Yes — match Claude's style
**Notes:** None

### Q4: Should AGENTS.md be a living document?

| Option | Description | Selected |
|--------|-------------|----------|
| Living — update per phase (Recommended) | AGENTS.md gets updated as new conventions emerge. Codex always has current context. Managed by GSD phase transitions. | ✓ |
| Fixed — set once | Write it once in Phase 1, don't touch it again. Simpler but may drift from reality over time. | |
| You decide | Claude determines the right approach based on how much the project evolves. | |

**User's choice:** Living — update per phase (Recommended)
**Notes:** None

---

## Routing Boundaries

### Q1: What types of tasks should be routed to Codex?

| Option | Description | Selected |
|--------|-------------|----------|
| Clearly-defined implementation | Writing new code from a spec or plan where the what-to-build is explicit. Codex's strongest use case. | ✓ |
| Test generation | Writing unit/integration tests for existing code. Codex is fast and cheap at this. | ✓ |
| Bulk file operations | Repetitive edits across many files (renaming, updating imports, applying patterns). Codex handles 1M context. | ✓ |
| Code review / diff analysis | Reviewing git diffs for bugs, style issues, or missed requirements. Research shows cross-model review catches more. | ✓ |

**User's choice:** All four selected
**Notes:** None

### Q2: How should the hook decide what's 'clearly-defined' vs 'complex reasoning'?

| Option | Description | Selected |
|--------|-------------|----------|
| Tool-call based (Recommended) | Route by WHAT Claude is doing: Write/Edit calls during plan execution → Codex. Architecture prompts, debugging, exploratory work → stays with Opus. Uses existing hook matcher pattern. | ✓ |
| GSD context based | Only route when inside a GSD execution phase with a plan. If there's no plan, everything stays with Opus. More conservative. | |
| Explicit flag based | Opus explicitly marks tasks for Codex in plan files (e.g., 'codex: true'). Most controlled but requires plan format changes. | |

**User's choice:** Tool-call based (Recommended)
**Notes:** None

### Q3: Should routing be opt-in or opt-out?

| Option | Description | Selected |
|--------|-------------|----------|
| Opt-in (Recommended) | Routing is OFF by default. Enable per project via .claude/settings.json or a config flag. Safest for the first version. | ✓ |
| Opt-out | Routing is ON by default for all projects. Disable per project if needed. More aggressive. | |
| This project only | Routing only works in the Claude_X_Codex project during development. Expand later manually. | |

**User's choice:** Opt-in (Recommended)
**Notes:** None

### Q4: When Codex finishes a routed task, how should the result come back to you?

| Option | Description | Selected |
|--------|-------------|----------|
| Silent — Opus summarizes | Codex output is injected as context for Opus. Opus presents the result in its own words. You never see raw Codex output. | |
| Attributed — labeled source | Opus presents the result but notes 'Codex handled this implementation'. You know which model did the work. | ✓ |
| You decide | Claude picks based on what makes the experience smoothest. | |

**User's choice:** Attributed — labeled source
**Notes:** None

---

## Claude's Discretion

- Failure behavior (Codex timeout/failure handling)
- Token log detail level
- Hook script architecture
- Security implementation details

## Deferred Ideas

None — discussion stayed within phase scope.
