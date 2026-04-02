# Retrospective: Claude X Codex

## Milestone: v1.0 — Claude X Codex

**Shipped:** 2026-04-02
**Phases:** 4 | **Plans:** 8 | **Tasks:** 14
**Timeline:** 1 day (single session)
**Commits:** 40 | **Lines added:** 8,694

### What Was Built

- Codex execution wrapper with 300s timeout, SIGTERM/SIGKILL, and JSONL token parsing
- AGENTS.md project brief with hard-stop architectural decision rule
- PreToolUse advisory routing hook (opt-out, v2.0) for global Codex task suggestions
- Stop hook review gate with ALLOW/BLOCK protocol and task-type-aware depth variation
- GSD wave-boundary validation with non-blocking background workers and sentinel files
- Multi-round Opus-Codex plan review loop (constructive + adversarial) with early exit
- Typed handoff specs with `decisions_not_taken` section and review state persistence
- Superpowers GPT-5.4-mini routing via skill override and plan review integration
- Session cost reporter generating savings reports vs Opus-only baseline

### What Worked

- **Security-first build order** — Phase 1 established all plumbing (auth, logging, routing) before any review logic was activated. No "move fast and fix later" shortcuts.
- **Hook-based integration** — Zero plugin source modifications. All 7 hooks use the native Claude Code hooks API. Survives plugin updates.
- **Worktree isolation for executors** — Parallel executor agents in git worktrees prevented merge conflicts and hook contention.
- **Plan checker verification loop** — Caught real issues (D-07 enforcement missing, D-12 depth variation missing) before execution. Saved rework.
- **Shared module architecture** — `codex-multi-round-reviewer.js` reused by both GSD and Superpowers hooks. Single implementation, two consumers.

### What Was Inefficient

- **Research agents on simple phases** — Phase 4 (Cost Reporting) had only 2 requirements and well-understood domain. Research was skipped, saving time. Should have been the default for all simple phases.
- **Some SUMMARY.md files missing `requirements_completed` frontmatter** — 3 of 8 summaries lacked this field, complicating the 3-source cross-reference during milestone audit.
- **AppArmor sandbox issue not caught until manual testing** — The bubblewrap sandbox failure (`apparmor_restrict_unprivileged_userns`) blocked all Codex CLI execution. Should have been detected in Phase 1 verification.

### Patterns Established

- **Fail-open for all hooks** — Every hook wraps in try/catch with `process.exit(0)`. Never blocks the user on hook failure.
- **Advisory routing (not auto-delegation)** — Opus sees routing suggestions and decides. Prevents cost runaway.
- **Token logging at every invocation site** — Each hook that calls Codex appends its own JSONL record. No central logging bottleneck.
- **Dual-mode scripts** — Hook scripts work both as hooks (stdin JSON) and standalone CLI tools (TTY detection). Enables manual testing.

### Key Lessons

1. **Plan verification catches real issues** — The checker found 2 blockers in Phase 2 plans that would have caused failed execution. The cost of verification (~90s) is far less than re-execution.
2. **Infrastructure phases should skip discuss** — Pure infrastructure with technical-only success criteria doesn't benefit from grey area proposals.
3. **System-level issues need early validation** — AppArmor restrictions, NODE_PATH for global requires, and plugin update durability were all discovered during execution. A "Wave 0" validation step would catch these earlier.
4. **User-space skill overrides are durable** — `~/.claude/skills/` correctly shadows plugin cache for Superpowers. This pattern works for any plugin modification.

### Cost Observations

- Model mix: 100% Opus for orchestration, Sonnet for research/execution/verification agents
- Codex CLI: all hook-triggered reviews use ChatGPT Plus subscription (no API cost)
- API: GPT-5.4-mini only for Superpowers parallel dispatch (minimal cost)
- Functional test showed 86.7% savings ($0.0233 actual vs $0.1759 Opus baseline)

## Cross-Milestone Trends

| Metric | v1.0 |
|--------|------|
| Phases | 4 |
| Plans | 8 |
| Tasks | 14 |
| Days | 1 |
| Requirements | 26/26 |
| Tech Debt Items | 4 (all resolved) |
| Verification Pass Rate | 75% first pass (Phase 2 needed revision) |
