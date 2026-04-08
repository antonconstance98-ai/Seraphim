# Phase 13: Codex Execution Pipeline - Context

**Gathered:** 2026-04-03
**Status:** Ready for planning

<domain>
## Phase Boundary

Transform the gsd-executor from an Opus-powered code writer into a thin orchestrator that generates handoff specs and delegates actual code writing to Codex CLI (free via subscription) with MiniMax API as fallback when Codex is rate-limited. This is the biggest cost-saving change in v2.0 — moves all code generation off Opus tokens.

</domain>

<decisions>
## Implementation Decisions

### Handoff spec format
- **D-01:** Natural language task description + file paths. The handoff spec tells Codex/MiniMax WHAT to do in plain English: "Create file X with function Y that does Z" or "Modify file X: change function Y to handle Z."
- **D-02:** Each handoff spec includes: task description, target file path(s), action (create/modify/delete), specification of changes, and a verification command (e.g., `grep for 'X' in the modified file`).
- **D-03:** For MiniMax fallback: the handoff spec also includes the current file content (since MiniMax has no filesystem access). MiniMax returns the full modified content as text, and the orchestrator writes it to disk.

### Executor model
- **D-04:** The gsd-executor thin orchestrator runs on **Sonnet 4.6 ($3/$15 per Mtok)**. It only reads plans, generates handoff specs, validates outputs, and manages git commits — no code writing. 10x cheaper than Opus for a mostly mechanical task.
- **D-05:** Set `executor_model` to `sonnet` in GSD model profile configuration. This is a GSD config change, not a hook change.

### Execution flow
- **D-06:** For each plan task, the orchestrator:
  1. Reads the task from PLAN.md
  2. Generates a natural language handoff spec
  3. Invokes Codex CLI: `codex exec --full-auto --json [spec]`
  4. Validates Codex output (check git diff, run verification command)
  5. Commits atomically per task (preserving GSD protocols)
  6. Writes SUMMARY.md after all tasks complete
- **D-07:** The orchestrator preserves ALL existing GSD protocols: atomic per-task commits, deviation handling, STATE.md updates, SUMMARY.md creation, checkpoint pausing.

### Fallback chain
- **D-08:** Codex CLI (free via subscription) → MiniMax API ($0.30/$1.20) → prompt user (fail-closed).
- **D-09:** Rate-limit detection via `runWithFallback()` from Phase 8: exit codes, stderr "rate limit"/"quota"/"usage limit", HTTP 429 in JSONL, timeout with no output, `rate_limit_pct >= 95`.
- **D-10:** When falling back to MiniMax: the orchestrator reads the target file(s), includes content in the handoff spec, sends to MiniMax API, receives modified content as text, and writes it to disk using the orchestrator's own Write/Edit tools.
- **D-11:** When both Codex and MiniMax fail: prompt user with both error messages. Options: (1) wait and retry Codex, (2) check MINIMAX_API_KEY, (3) skip this task. Fail-closed — never silently fall back to Opus for code writing.
- **D-12:** Log every fallback event to `token-log.jsonl` with `source: 'cli-fallback'` or `source: 'api-fallback'` so the dashboard can track fallback frequency.

### Claude's Discretion
- Handoff spec prompt template wording
- Validation depth (basic grep vs running tests)
- How to handle multi-file tasks (one Codex call per file or one call for all files)
- Whether to batch small tasks into a single Codex invocation for efficiency

</decisions>

<specifics>
## Specific Ideas

- The key insight is that Codex CLI runs locally with filesystem access (sandboxed), so it can create/modify files directly. MiniMax is API-only — it returns text, and the orchestrator must write files. Same handoff spec, different execution path.
- The orchestrator on Sonnet is cheap enough that even if it takes a few attempts to generate a good handoff spec, the total cost is still far below Opus writing code directly.
- This change alone could reduce daily spend from ~$10-15 (Opus writing code) to ~$1-3 (Sonnet orchestrating + Codex executing for free).

</specifics>

<canonical_refs>
## Canonical References

### Phase 8 Foundation (dependency)
- `.planning/phases/08-minimax-foundation/08-CONTEXT.md` — `runWithFallback()` fallback chain, `runMinimax()` interface

### Existing Execution Infrastructure
- `~/.claude/agents/gsd-executor.md` — Current gsd-executor agent definition (runs on Opus, writes code directly). Must be modified to become thin orchestrator.
- `~/.claude/get-shit-done/workflows/execute-phase.md` — Execute-phase workflow that spawns gsd-executor with `model="{executor_model}"`
- `~/.claude/get-shit-done/workflows/execute-plan.md` — Execute-plan workflow with Task() spawn pattern
- `~/.claude/get-shit-done/references/model-profiles.md` — GSD model profile configuration for `executor_model`

### Codex CLI
- `~/.claude/hooks/codex-exec.js` — `runCodexExec()` for spawning `codex exec --full-auto --json`

### Research
- `minimax-m2.7-synthesis.md` §6 "Codex Execution Pipeline" — Architecture diagram, fallback flow
- `minimax-m2.7-synthesis.md` §4 "Codex CLI vs MiniMax API" — Capability comparison table

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `gsd-executor.md` agent — Has complete execution flow (load plan, execute tasks, commit, write SUMMARY.md). The agent definition needs modification, not replacement.
- `codex-exec.js` — `runCodexExec(prompt, { cwd, timeoutMs, model })` for Codex CLI invocation.
- `minimax-exec.js` (Phase 8) — `runWithFallback()` for the complete fallback chain.

### Established Patterns
- GSD executor reads plan via `gsd-tools.cjs init execute-phase` JSON
- `executor_model` resolved from GSD model profile config at orchestration time
- Atomic commits via `gsd-tools.cjs commit` with `--no-verify` in parallel mode
- Deviation handling: auto-add missing functionality, skip impossible tasks, log deviations

### Integration Points
- `gsd-executor.md` agent definition — change from code-writing agent to spec-generating orchestrator
- `.planning/config.json` — set `executor_model` to `sonnet` (or via `/gsd:set-profile`)
- `execute-phase.md` workflow — `Task(subagent_type="gsd-executor", model="{executor_model}")` spawn. Model changes to Sonnet.
- Codex CLI invocation happens inside the gsd-executor subagent (Sonnet), not at the orchestrator level.

</code_context>

<deferred>
## Deferred Ideas

- Parallel Codex invocations for independent tasks within a plan — currently sequential per task, parallel later if bottleneck
- Smart handoff spec complexity detection (auto-switch between simple and detailed specs based on task difficulty)
- Codex warm-up (pre-spawn Codex process to reduce cold-start latency)

</deferred>

---

*Phase: 13-codex-execution-pipeline*
*Context gathered: 2026-04-03*
