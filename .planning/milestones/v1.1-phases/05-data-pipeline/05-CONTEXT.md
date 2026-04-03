# Phase 5: Data Pipeline - Context

**Gathered:** 2026-04-02
**Status:** Ready for planning
**Mode:** Auto-generated (infrastructure phase — discuss skipped)

<domain>
## Phase Boundary

The global aggregator runs standalone, correctly merges all per-project token logs into `global.jsonl`, handles null session IDs as "unattributed", and completes a repeat run in under 5 ms.

Requirements: PIPE-01, PIPE-02, PIPE-03, PIPE-04

</domain>

<decisions>
## Implementation Decisions

### Claude's Discretion
All implementation choices are at Claude's discretion — pure infrastructure phase. Use ROADMAP phase goal, success criteria, and codebase conventions to guide decisions.

Key research findings to incorporate:
- Use `fs.glob()` (Node.js v22 built-in) for project discovery — returns AsyncIterator, use `for await...of`
- Dedup key: `session_id + timestamp` (handles null session_id via `null|timestamp`)
- Discovery roots: configurable via `~/.claude/dashboard/config.json`, defaults include `~/projects`, `~/agent`, `~/gsd-workspaces`, `/mnt/hdd`
- Exclude paths matching `/.claude/worktrees/`, `/node_modules/`, `/.git/`
- mtime-gated incremental reads: skip files whose mtime hasn't changed since last run
- Append-only global.jsonl: never rewrite, only append new records
- Write-then-renameSync for atomic writes on shared files

</decisions>

<code_context>
## Existing Code Insights

### Reusable Assets
- `codex-exec.js` — contains `computeCost()` function for Codex model pricing
- `codex-cost-reporter.js` — established SessionStart hook pattern (stdin JSON, stdout additionalContext)
- `codex-token-logger.js` — JSONL schema reference (timestamp, session_id, model, source, task_type, tokens, cost_usd)

### Established Patterns
- All hooks use stdin JSON for event data, stdout JSON for additionalContext
- Silent fail: outer try/catch with process.exit(0) — never block session
- Timeout guard: setTimeout → process.exit(0) for stdin close protection

### Integration Points
- New script at `~/.claude/hooks/codex-global-aggregator.js`
- Reads from: `{project}/.planning/token-log.jsonl` (multiple projects)
- Writes to: `~/.claude/dashboard/global.jsonl`, `project-index.json`, `last-run.json`

</code_context>

<specifics>
## Specific Ideas

No specific requirements — infrastructure phase. Refer to ROADMAP phase description and success criteria.

</specifics>

<deferred>
## Deferred Ideas

None — infrastructure phase.

</deferred>
