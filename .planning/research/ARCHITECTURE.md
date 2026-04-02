# Architecture Patterns: Multi-Model Claude Code Integration

**Domain:** Claude Code plugin modification — multi-model orchestration (Opus + Codex)
**Researched:** 2026-04-02
**Sources:** Live codebase inspection (GSD plugin, Superpowers plugin, Codex plugin), official Claude Code hooks documentation, existing model comparison research

---

## Recommended Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    CLAUDE CODE SESSION                          │
│                 (Opus 4.6 — primary context)                    │
│                                                                 │
│  settings.json hooks                                            │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ PreToolUse  ─► guard scripts (block/allow/modify)       │   │
│  │ PostToolUse ─► context-monitor.js (inject warnings)     │   │
│  │ Stop        ─► stop-review-gate-hook.mjs (BLOCK/ALLOW)  │   │
│  │ SessionStart ─► gsd-check-update.js                     │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  GSD Plugin (.planning/ filesystem state)                       │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ Orchestrator (Opus) reads STATE.md / ROADMAP.md         │   │
│  │ Spawns: Task(subagent_type="gsd-executor", ...)         │   │
│  │ Spawns: Task(subagent_type="gsd-verifier", ...)         │   │
│  │ File handoff: PLAN.md ─► executor ─► SUMMARY.md        │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  Superpowers Plugin (skills system)                             │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ Skills loaded from ~/.claude/skills/ at startup         │   │
│  │ dispatching-parallel-agents: Task() x N in parallel     │   │
│  │ systematic-debugging: parallel hypothesis agents        │   │
│  │ requesting-code-review: spawns reviewer agents          │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└──────────────────────┬──────────────────────────────────────────┘
                       │ spawnSync / spawn (Node.js child_process)
                       ▼
┌─────────────────────────────────────────────────────────────────┐
│               CODEX PLUGIN RUNTIME LAYER                        │
│         (codex-companion.mjs — the bridge process)              │
│                                                                 │
│  Commands: setup | review | adversarial-review | task |        │
│            task-worker | status | result | cancel              │
│                                                                 │
│  Job state: $CLAUDE_PLUGIN_DATA/state/{workspace-hash}/        │
│   ├── state.json       (config: stopReviewGate)               │
│   └── jobs/            (per-job JSON files)                    │
│        └── {job-id}.json (queued/running/completed/cancelled)  │
│                                                                 │
│  Execution modes:                                               │
│   ├── Foreground: spawnSync → blocks caller until done         │
│   ├── Background: spawn detached → worker process, polled      │
│   └── Stop gate: blocking hook (up to 15 min timeout)          │
│                                                                 │
└──────────────────────┬──────────────────────────────────────────┘
                       │ JSON-RPC over Unix socket (app-server broker)
                       │ CODEX_COMPANION_APP_SERVER_ENDPOINT env var
                       ▼
┌─────────────────────────────────────────────────────────────────┐
│              CODEX APP SERVER (broker process)                  │
│         OpenAI Codex CLI — @openai/codex npm package           │
│                                                                 │
│  Protocol: JSON-RPC 2.0 over Unix socket                       │
│  Methods: thread/start, thread/resume, turn/start,             │
│           turn/interrupt, review/start, thread/list            │
│                                                                 │
│  Sandbox modes: read-only | workspace-write                    │
│  OS-level sandboxing: Seatbelt (macOS) / Landlock (Linux)     │
│  Reasoning effort: none|minimal|low|medium|high|xhigh          │
│  Model selection: gpt-5.4 | gpt-5.3-codex | gpt-5.3-codex-spark│
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Component Boundaries

### Component 1: Claude Code Session (Opus 4.6)

**Responsibility:** Primary orchestrator — all architectural decisions, planning, complex reasoning, and agent coordination. Reads and writes `.planning/` filesystem state. Spawns subagents via `Task()` tool.

**Communicates with:**
- Hook scripts (via hooks system — JSON on stdin, JSON on stdout)
- GSD subagents (via `Task()` spawning — blocking, returns result)
- Superpowers skills (loaded into context, shape behavior)
- Codex plugin (via `Bash(node ... codex-companion.mjs ...)` calls)

**Does NOT:**
- Make low-level tool calls inside Codex CLI
- Directly call OpenAI API for execution tasks
- Manage Codex job state files

---

### Component 2: Hook Scripts (Node.js processes)

**Responsibility:** Intercept Claude's tool use and session lifecycle events. Can inject context Claude sees, block/allow tool calls, or trigger side effects (like a Codex review).

**Communicates with:**
- Claude Code: stdin/stdout JSON protocol
- Filesystem: reads/writes `/tmp/claude-ctx-{session_id}.json` for metrics
- Codex companion: the Stop hook calls `codex-companion.mjs task` via `spawnSync`

**Existing hooks in production:**
- `gsd-context-monitor.js` — PostToolUse — injects context warnings when token window fills
- `gsd-prompt-guard.js` — PreToolUse — advisory scan of `.planning/` writes for injection patterns
- `claude-settings-guard.js` — PreToolUse — guards claude settings files
- `stop-review-gate-hook.mjs` — Stop — can BLOCK Claude from stopping (calls Codex synchronously, waits up to 15 minutes, returns `{ decision: "block", reason: "..." }` or passes)

**Key protocol facts (verified from official docs):**
- Hook input arrives on stdin as JSON
- `{ decision: "block", reason: "..." }` on stdout prevents the event (Stop hooks use this to keep session running)
- `{ hookSpecificOutput: { hookEventName: "...", additionalContext: "..." } }` injects text Claude sees
- `{ hookSpecificOutput: { hookEventName: "PreToolUse", permissionDecision: "deny" } }` blocks tool calls
- Exit code 2 = blocking error (stderr shown to user, stdout ignored)
- Token usage data is NOT available in hook input (confirmed in official docs)

---

### Component 3: GSD Plugin (Filesystem Orchestration)

**Responsibility:** Phase-based project execution. Breaks work into PLAN.md files, spawns specialized subagents (`gsd-executor`, `gsd-verifier`, etc.), tracks state in `.planning/STATE.md`, and manages wave-based parallelism.

**Communicates with:**
- Opus orchestrator: PLAN.md / SUMMARY.md / STATE.md files (filesystem handoff)
- Subagents: `Task(subagent_type="gsd-executor", model="...", isolation="worktree", prompt="...")` — blocks until complete
- `gsd-tools.cjs` binary: CLI tool for state management operations

**Subagent spawning pattern (from execute-phase.md source):**
```
Task(
  subagent_type="gsd-executor",
  model="{executor_model}",
  isolation="worktree",        // isolated git worktree per agent
  prompt="... @path/to/PLAN.md ..."
)
```

**Key architectural constraint:** Each subagent gets a fresh 200k-1M token context window. Orchestrator stays lean (~10-15% of context). Subagents read their plan files directly — paths are passed, not content.

---

### Component 4: Superpowers Plugin (Skills System)

**Responsibility:** Behavioral skills that shape how Opus executes tasks. Skills are loaded from `~/.claude/skills/` at session start and activated by name or matching description. Skills for parallel agents, systematic debugging, code review, and plan execution are the most relevant.

**Communicates with:**
- Claude context: skills inject into the agent's instruction set
- Codex (via skill rules): `dispatching-parallel-agents` skill includes model routing hints — `model: "claude-opus-4-0"` for judgment tasks, `model: "claude-sonnet-4-5"` for mechanical tasks
- Codex CLI for Superpowers: `~/.agents/skills/` — Codex CLI natively scans this directory and loads SKILL.md files (separate path from Claude Code's `~/.claude/skills/`)

**Key fact for Codex integration:** Superpowers already has a path (`docs/README.codex.md`) for running skills inside Codex CLI via `~/.agents/skills/` symlink. This is the injection point for adding Superpowers-compatible skills that run when Codex executes.

---

### Component 5: Codex Companion Runtime (codex-companion.mjs)

**Responsibility:** The bridge between Claude Code and the Codex CLI. Manages jobs (queued/running/completed state), provides foreground and background execution modes, handles review and adversarial review workflows, and exposes a CLI interface callable from hooks and Bash tool calls.

**Communicates with:**
- Claude Code: called via `Bash(node ... codex-companion.mjs <subcommand> <args>)` tool calls
- Codex App Server: JSON-RPC over Unix socket (`CODEX_COMPANION_APP_SERVER_ENDPOINT` env var)
- Job state files: `$CLAUDE_PLUGIN_DATA/state/{workspace-hash}/jobs/`
- Stop hook: called via `spawnSync` from `stop-review-gate-hook.mjs`

**Available subcommands:**
- `task [--background] [--write] [--model X] [--effort X] [prompt]` — run a task in Codex
- `review [--wait|--background] [--scope auto|working-tree|branch]` — native Codex reviewer
- `adversarial-review [focus text]` — adversarial review with custom prompt
- `status [job-id] [--wait]` — check job status (polls until complete with `--wait`)
- `result [job-id]` — fetch completed job output
- `cancel [job-id]` — interrupt a running job

**Execution model:** Background tasks spawn a detached `task-worker` child process. The worker writes job state to disk. Polling via `status --wait` checks the file. No streaming — async handoff.

---

### Component 6: Codex App Server (Broker Process)

**Responsibility:** The actual Codex CLI process managed by the companion. Runs Codex turns in sandboxed contexts. Exposes JSON-RPC methods for starting threads, running turns, and interrupting in-progress work.

**Communicates with:**
- Codex companion: JSON-RPC over Unix socket
- OpenAI API: outbound HTTPS calls for model inference
- Local filesystem: reads/writes files in sandbox modes (read-only or workspace-write)

**Thread model:** Each `task` call either starts a new thread or resumes an existing one (via `--resume-last`). Thread IDs are stored in job files, enabling multi-turn conversations with Codex across Claude sessions.

---

## Data Flow: Opus-Codex Plan Review Loop

This is the core interaction pattern the project needs to build. Based on how the existing Codex plugin handles review:

```
1. OPUS generates plan document
   └── Writes to: .planning/phases/XX-name/XX-YY-PLAN.md

2. REVIEW TRIGGER (hook or explicit command)
   └── Options:
       a. PostToolUse hook fires on Write to .planning/**-PLAN.md
          ─► hook calls: codex-companion.mjs task --json "[review prompt]"
          ─► OR: hook injects additionalContext telling Opus to trigger review
       b. GSD workflow step explicitly calls Bash(codex-companion.mjs task ...)
       c. Opus reads plan, constructs prompt, calls Bash directly

3. HANDOFF SPEC written to file (recommended pattern)
   └── File: .planning/codex-handoff/{plan-id}-handoff.md
       Content: objective, constraints, files to touch, success criteria
       Why file: avoids token overhead of passing large plans in prompt string

4. CODEX receives task
   └── codex-companion.mjs task --write "[read .planning/codex-handoff/X.md and respond]"
   └── Codex reads handoff file, writes response to stdout (captured as rawOutput)

5. CODEX response captured
   └── codex-companion.mjs exits → stdout contains JSON { rawOutput, status, touchedFiles }
   └── If background: poll with `codex-companion.mjs status --wait {job-id}`
       then: `codex-companion.mjs result {job-id}`

6. OPUS reads Codex response
   └── Options:
       a. Codex writes response file: .planning/codex-responses/{plan-id}-review.md
          Opus reads it with Read tool
       b. Hook captures stdout and injects as additionalContext (for short responses)
       c. Bash tool returns output inline (for foreground calls)

7. OPUS incorporates feedback
   └── Updates PLAN.md with revisions
   └── Round 2: repeat steps 2-6 (2-3 rounds total per research findings)

8. PLAN APPROVED
   └── Opus marks plan as reviewed, proceeds to execution
```

**Confidence:** HIGH — based on actual codex-companion.mjs source code and Stop hook pattern

---

## Data Flow: Token Tracking

Token usage is NOT exposed in hook input (confirmed by official docs). The viable architectures are:

```
Option A: Wrapper Script (Recommended)
─────────────────────────────────────
Before calling codex-companion.mjs:
  - Record timestamp, task type, model

After call returns:
  - Parse stdout for rawOutput length (proxy for output tokens)
  - Append to: .planning/token-log.jsonl
    { timestamp, task_type, model, approx_output_tokens, estimated_cost, session_id }

Tool: A thin bash wrapper script wrapping every codex-companion.mjs call


Option B: PostToolUse Hook (Opus side)
──────────────────────────────────────
Fires after every Bash tool use.
Inspect tool_input.command for "codex-companion.mjs"
Read most recent job result file from job state dir
Extract rawOutput length → estimate tokens
Write to .planning/token-log.jsonl


Option C: Modified codex-companion.mjs
───────────────────────────────────────
Add --log-tokens flag to codex-companion.mjs
After each run, append to token log file
(Requires modifying plugin source — within project scope)
```

**Recommendation:** Option C (modify codex-companion.mjs) for accuracy; Option A as fallback.
Option C is within scope — project explicitly modifies plugin source.

---

## Data Flow: GSD Hook Integration Points

Where the Codex review loop can hook into GSD's existing workflow:

```
GSD workflow point               Hook opportunity
─────────────────────────────────────────────────────────────
plan-phase completes             PostToolUse on Write to *-PLAN.md
                                 → trigger Codex adversarial review of the plan

execute-phase begins each wave   Between wave N and wave N+1
                                 → Codex cross-plan integration check

gsd-verifier creates VERIFY.md   PostToolUse on Write to *-VERIFICATION.md
                                 → Codex secondary validation pass

Stop hook (session end)          Already exists (stop-review-gate-hook.mjs)
                                 → Currently: reviews git changes
                                 → Enhanced: adversarial review of last Opus turn
```

---

## Architecture Patterns to Follow

### Pattern 1: File-Based Handoff (HIGH confidence)
**What:** Opus writes a structured handoff file; Codex reads it. Model-to-model communication via filesystem, not inline prompt strings.

**When:** Any task where the specification is longer than ~500 tokens. Avoids shell escaping issues, allows Codex to use its Read tool on the file, enables version control of handoff specs.

**Example structure:**
```markdown
# Codex Handoff: Plan Review — Phase 2 Plan 03

## Task
Review the implementation plan below for correctness, completeness, and simplicity.
Return: APPROVED | REVISE: [issues]

## Plan Summary
[extracted from PLAN.md — not full file, just key sections]

## Review Focus
- Are the file modifications minimal and non-breaking?
- Does the approach match the stated objective?
- Are there simpler alternatives worth noting?
```

### Pattern 2: Background-Then-Poll (MEDIUM confidence)
**What:** Trigger Codex in background (`--background` flag), continue Opus work, then poll for result before proceeding to next step.

**When:** Codex review/validation tasks that don't block the critical path — e.g., Superpowers parallel verification, background validation during wave execution.

**Implementation:** `codex-companion.mjs task --background "[prompt]"` returns `{jobId, status: "queued"}`. Later: `codex-companion.mjs status --wait {jobId}` blocks until complete.

### Pattern 3: Stop Hook as Quality Gate (HIGH confidence — already in production)
**What:** The Stop hook fires when Opus finishes a response. A blocking decision (`{ decision: "block", reason: "..." }`) forces Opus to continue and address the issues.

**When:** End-of-turn review gate — Codex reviews the last Opus turn and can force additional work.

**Existing example:** `stop-review-gate-hook.mjs` uses `parseStopReviewOutput()` — Codex returns `ALLOW: ...` or `BLOCK: [reason]` as first line of output. This protocol should be reused.

### Pattern 4: PostToolUse Injection (HIGH confidence)
**What:** PostToolUse hook fires after Bash/Write/Edit, runs lightweight logic, and injects `additionalContext` that Opus sees before its next turn.

**When:** Triggering Codex review automatically when Opus writes a plan file. Hook detects the write target path, launches Codex in background, injects a note telling Opus to wait for and incorporate the review.

---

## Anti-Patterns to Avoid

### Anti-Pattern 1: Inline Prompt Stuffing
**What:** Passing full PLAN.md content as a string argument to `codex-companion.mjs task "...entire plan here..."`.

**Why bad:** Shell escaping breaks, argument length limits, logs expose plan content, no way to version-control the handoff. The existing codex-rescue agent agent already passes prompts via positional arg, but that's for short prompts only.

**Instead:** Write `.planning/codex-handoff/{id}.md`, pass path reference.

### Anti-Pattern 2: Synchronous Review in Every Hook
**What:** Making every PostToolUse hook call Codex synchronously via `spawnSync`.

**Why bad:** Every tool use (Bash, Edit, Write, Grep, Glob) fires PostToolUse. Synchronous Codex calls in a hook block the entire session for every single tool use. The `stop-review-gate-hook.mjs` does use `spawnSync` but only on the Stop event (rare) with a 15-minute timeout.

**Instead:** Background Codex calls for non-critical reviews. Only use synchronous/blocking for explicit quality gates (Stop hook, phase transition checkpoints).

### Anti-Pattern 3: Bypassing the Companion Script
**What:** Calling `codex` CLI directly from hooks or Bash tool calls instead of going through `codex-companion.mjs`.

**Why bad:** The companion handles job state persistence, background worker processes, thread ID tracking (enabling multi-turn), error handling, and the app-server broker lifecycle. Bypassing it loses all state tracking needed for status polling and result retrieval.

**Instead:** Always route through `node "${CLAUDE_PLUGIN_ROOT}/scripts/codex-companion.mjs"`.

### Anti-Pattern 4: Token Tracking via Polling Loop
**What:** A hook that repeatedly checks job state files to accumulate token counts across a session.

**Why bad:** Hooks are stateless single-invocation processes. State between hook invocations must go through files. A polling loop inside a hook would block the session.

**Instead:** Append-only log file written after each Codex call completes. Session report reads the log at end of session.

### Anti-Pattern 5: Letting Codex Make Architectural Decisions
**What:** Giving Codex `--write` access and open-ended prompts like "improve this codebase" without a constrained handoff spec.

**Why bad:** Research confirms Codex proceeds with assumptions rather than seeking clarification, more frequently invents APIs at scale, and drifts in long sessions. The value of Codex is fast execution of well-specified tasks.

**Instead:** Opus always generates the spec. Codex always executes against the spec. Codex reviews are for correctness and simplicity, not architectural direction.

---

## Component Build Order

Based on dependencies between components:

```
Phase 1: Foundation — Hook + Routing Infrastructure
  1a. Token tracking wrapper / log schema
      (needed early — all later phases depend on it for cost validation)
  1b. Codex handoff file format + directory structure
      (.planning/codex-handoff/, .planning/codex-responses/)
  1c. Routing decision logic
      (simple bash script or Node module: given task type → model + flags)

Phase 2: GSD Integration Points
  2a. PostToolUse hook — trigger Codex review on PLAN.md write
      (depends on: 1b handoff format, 1a token tracking)
  2b. Execute-phase workflow modification — Codex cross-wave check
      (depends on: 1b, routing logic from 1c)
  2c. Verification workflow modification — Codex secondary validation
      (depends on: 2a pattern established)

Phase 3: Superpowers Integration Points
  3a. Parallel agent skill modification — model routing hints
      (depends on: Phase 1 routing logic)
  3b. Parallel hypothesis testing via Codex background tasks
      (depends on: Phase 2 patterns working)
  3c. Parallel code review — Codex + Opus dual reviewer
      (depends on: 3a)

Phase 4: Opus-Codex Review Loop
  4a. Review loop command / workflow step
      (depends on: all Phase 1, GSD integration from Phase 2)
  4b. 2-3 round review cycle with convergence detection
  4c. Adaptive handoff spec generation (file-level vs feature-level)

Phase 5: Token Tracking Reports
  5a. Session report generator (reads token-log.jsonl)
      (depends on: token tracking from Phase 1)
  5b. Cost comparison baseline (vs Opus-only)
```

**Why this order:**
- Token tracking must come first — it's the success metric for the entire project
- GSD integration before Superpowers — GSD has clearer extension points (workflow files), Superpowers skills are more fragile
- Review loop last — it synthesizes all the infrastructure built in earlier phases

---

## Scalability Considerations

| Concern | Current State | With Integration | Mitigation |
|---------|---------------|-----------------|------------|
| Hook latency | Hooks must complete quickly (hooks run on every tool use) | PostToolUse Codex call could add 10-60s per tool use | Background Codex calls only; never synchronous in PostToolUse |
| Concurrent jobs | Codex companion supports background job queue | Multiple parallel waves could trigger multiple Codex reviews simultaneously | Job state files handle this; use `status --wait` for synchronization |
| Token log growth | Not applicable | Append-only JSONL can grow large in long sessions | Rotate by session; archive old logs |
| Context window (Opus) | GSD already manages this (10-15% orchestrator budget) | Codex review results injected via additionalContext or file reads | Keep Codex responses concise; use file reads not inline injection for large reviews |
| Codex thread reuse | Not applicable | Multi-turn threads via `--resume-last` reduce cold-start overhead | Use thread persistence for related review rounds |

---

## Confidence Assessment

| Area | Confidence | Source |
|------|------------|--------|
| Hook system protocol (PreToolUse/PostToolUse/Stop) | HIGH | Live source code (gsd-context-monitor.js, stop-review-gate-hook.mjs) + official docs |
| Codex companion invocation pattern | HIGH | Live source code (codex-companion.mjs, codex-rescue.md agent) |
| Codex app-server JSON-RPC protocol | HIGH | Live source (app-server.mjs, app-server-protocol.d.ts) |
| GSD subagent spawning (Task() API) | HIGH | Live source (execute-phase.md workflow) |
| Superpowers parallel agent pattern | HIGH | Live source (dispatching-parallel-agents SKILL.md) |
| Token tracking via hook | MEDIUM | Official docs confirm no token data in hook input; wrapper approach is inferred |
| Review loop round-trip timing | MEDIUM | Known Codex is fast (240+ tok/s), but total round-trip depends on task size |
| Adaptive handoff spec complexity | LOW | Design decision not yet validated; file-level vs feature-level distinction from PROJECT.md only |

---

## Sources

- Live codebase: `/home/alucard/.claude/plugins/marketplaces/openai-codex/plugins/codex/scripts/codex-companion.mjs`
- Live codebase: `/home/alucard/.claude/plugins/marketplaces/openai-codex/plugins/codex/scripts/stop-review-gate-hook.mjs`
- Live codebase: `/home/alucard/.claude/plugins/marketplaces/openai-codex/plugins/codex/scripts/lib/app-server.mjs`
- Live codebase: `/home/alucard/.claude/hooks/gsd-context-monitor.js`
- Live codebase: `/home/alucard/.claude/get-shit-done/workflows/execute-phase.md`
- Live codebase: `/home/alucard/.claude/plugins/cache/claude-plugins-official/superpowers/5.0.7/skills/dispatching-parallel-agents/SKILL.md`
- Official: [Claude Code Hooks Reference](https://code.claude.com/docs/en/hooks) — hook events, JSON protocol, decision control
- Research: `docs/research/opus-vs-codex-model-comparison.md` — task routing decisions and cross-model review patterns
