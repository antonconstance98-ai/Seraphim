# Phase 1: Foundation - Research

**Researched:** 2026-04-02
**Domain:** Claude Code hooks + Codex CLI integration, security hardening, token tracking
**Confidence:** HIGH

---

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions

**Codex Briefing (AGENTS.md)**
- D-01: Guardrails are moderate — Codex can make minor judgment calls (naming, small refactors) but must escalate anything structural back to Opus
- D-02: AGENTS.md includes a full project brief — project description, tech stack, file structure conventions, security rules, and the hard rule that Opus is the sole architect
- D-03: AGENTS.md matches Claude's communication style — plain English explanations, no jargon, simple solutions. Consistent experience across both models
- D-04: AGENTS.md is a living document — updated at GSD phase transitions as new conventions emerge. Codex always has current project context

**Routing Boundaries**
- D-05: Codex handles all four task types: clearly-defined implementation, test generation, bulk file operations, and code review / diff analysis
- D-06: Routing detection is tool-call based — route by what Claude is doing (Write/Edit calls during plan execution go to Codex; architecture, debugging, exploratory work stays with Opus). Uses existing hook matcher pattern
- D-07: Routing is opt-in — OFF by default, enabled per project via `.claude/settings.json` or a config flag. Safest for first version
- D-08: Results are attributed — Opus presents the result but notes which model did the work. User always knows when Codex handled something

### Claude's Discretion
- Failure behavior: Claude decides how to handle Codex timeouts/failures (silent fallback vs notification) based on what works best for the experience
- Token log detail level: Claude decides the right granularity for `.planning/token-log.jsonl` based on what Phase 4 cost reporting needs
- Hook script architecture: Claude follows existing `~/.claude/hooks/` patterns (Node.js, stdin JSON, `additionalContext` output)
- Security implementation: Claude handles version verification, env var protection, and SUBPROCESS_ENV_SCRUB setup

### Deferred Ideas (OUT OF SCOPE)
None — discussion stayed within phase scope.
</user_constraints>

---

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| FNDTN-01 | AGENTS.md spec file exists at repo root, giving Codex project context, conventions, and the hard rule that Codex never makes architectural decisions | AGENTS.md format well-documented; official pattern confirmed by codex-plugin-cc source |
| FNDTN-02 | Codex CLI can be invoked from Claude Code hooks via `codex exec --json` with timeout wrapper (300s max) | `--json` flag confirmed on v0.118.0; timeout wrapper pattern established; codex-companion.mjs is the right bridge |
| FNDTN-03 | PreToolUse hook intercepts Claude tool calls and routes Codex-appropriate tasks to Codex CLI | Existing hook pattern (gsd-context-monitor.js) is the template; `matcher: "Write|Edit"` field routes by tool |
| FNDTN-04 | Claude Code security is verified (version 2.0.65+ for CVE patches, API keys in env vars only) | Version 2.1.90 confirmed — satisfies requirement; `CLAUDE_CODE_SUBPROCESS_ENV_SCRUB=1` not yet set — needs action |
| FNDTN-05 | Headless Codex CLI authentication works via API key (not ChatGPT web login) for hook-triggered invocations | Current auth is chatgpt OAuth mode; `OPENAI_API_KEY` env var not set — needs action before hooks go live |
| ROUT-01 | Opus remains sole model for architectural decisions — enforced by hooks and AGENTS.md, not just convention | AGENTS.md hard-rule pattern; PreToolUse hook can inject advisory context; routing logic in hook is the enforcement point |
| ROUT-03 | Fallback routing gracefully degrades to Opus when Codex CLI rate-limits or fails | Failure handler in hook script; fail-CLOSED pattern (prompt user, not auto-route) recommended by research |
| ROUT-04 | Codex CLI (subscription) preferred over API calls; API used only for quick model-to-model communication | Confirmed: CLI uses ChatGPT Plus ($20/mo subscription); API (`openai@6.33.0`) for short advisory queries only |
| TRCK-01 | Every model call logged with: model name, task type, tokens_in, tokens_out, cost, timestamp | Codex session JSONL confirms exact token fields; OpenAI SDK response.usage fields confirmed |
| TRCK-02 | Token tracking covers both Claude (from session JSONL) and Codex (from `--json` JSONL output) | Both schemas verified from live files; separate per-provider token types required |
</phase_requirements>

---

## Summary

Phase 1 builds the integration plumbing that all subsequent phases depend on. Every tool it needs is already installed on this machine — Node.js v22.22.0, Codex CLI v0.118.0, the official `codex-plugin-cc` plugin (v1.0.2), and the OpenAI Node.js SDK (v6.33.0). No new runtimes or frameworks are needed; the only new package to install is `openai` for the API fallback path, and that is already present globally.

The work divides into four clusters that must be done in this order: (1) security baseline — set `CLAUDE_CODE_SUBPROCESS_ENV_SCRUB=1` and store `OPENAI_API_KEY` for hook-level auth; (2) AGENTS.md authoring — the project brief Codex reads before every task; (3) hook infrastructure — a PreToolUse routing hook and a token-logging PostToolUse hook, both following the established Node.js stdin/stdout JSON pattern; and (4) token log schema — the append-only JSONL file that Phase 4 cost reporting depends on. All four clusters are Phase 1 prerequisites; none is optional.

The highest-risk item is the Codex CLI silent hang bug (GitHub issues #14303, #14314, #13708), which is an open unresolved bug in v0.118.0. Every `codex exec` call from a hook must be wrapped with `timeout 300` to prevent the Claude Code session from hanging indefinitely. The second-highest risk is API key exfiltration via hook scripts (CVE-2025-59536); `CLAUDE_CODE_SUBPROCESS_ENV_SCRUB=1` must be set in `~/.bashrc` before any hook that has access to API keys is activated.

**Primary recommendation:** Build in this order — security setup first, then AGENTS.md, then the routing hook (opt-in OFF by default), then the token logger. Verify each layer works before building the next.

---

## Standard Stack

### Core

| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| Node.js | v22.22.0 (installed) | Hook scripts, all orchestration logic | All existing GSD/Superpowers hooks are Node.js; matching runtime eliminates toolchain friction |
| `@openai/codex` CLI | 0.118.0 (installed at `~/.npm-global/bin/codex`) | Non-interactive Codex execution from hooks | Installed and working; uses ChatGPT Plus subscription (no API billing for execution tasks) |
| `codex-companion.mjs` | Part of `codex@openai-codex` v1.0.2 | Official bridge to Codex App Server | Manages job state, background/foreground execution, thread persistence; always route through this, never raw CLI |
| Claude Code hooks | v2.1.90 (current) | Lifecycle event triggers for hook scripts | Native to Claude Code; no third-party dependency; already used by GSD hooks |
| `openai` npm package | 6.33.0 (globally installed) | Direct API calls for short advisory queries | Lower overhead than spawning a CLI process for 1-2 sentence responses; already installed |

### Supporting

| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| `codex-plugin-cc` | 1.0.2 (installed in Claude Code) | Official `/codex:review`, `/codex:rescue` commands | Use for manual review triggers; hooks reference `codex-companion.mjs` from this plugin |
| `fs` / `path` / `child_process` | Node.js built-ins | File I/O, subprocess management in hooks | No external dependency needed for hook scripts |

### Alternatives Considered

| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| `codex exec --json` via `codex-companion.mjs` | Raw `codex exec` CLI subprocess | `codex-companion.mjs` adds job state tracking, app-server protocol, and thread management for free. Raw CLI is simpler but loses resume/status capability |
| `openai` SDK for advisory calls | `codex exec` CLI for all calls | CLI startup is 1-2s; too slow for synchronous `additionalContext` injection in PreToolUse hooks. SDK is faster for short responses |
| ChatGPT subscription auth (current) | `OPENAI_API_KEY` env var | OAuth tokens can expire and need browser refresh. API key is more reliable for hooks running at odd hours. FNDTN-05 requires API key mode |

**Installation:**
```bash
# Already installed — verify, don't reinstall
node --version          # v22.22.0
codex --version         # codex-cli 0.118.0
npm show openai version # 6.33.0

# Only action needed: set env vars in ~/.bashrc
# export OPENAI_API_KEY="sk-..."        # for FNDTN-05 and API fallback
# export CLAUDE_CODE_SUBPROCESS_ENV_SCRUB=1  # for FNDTN-04 security
```

**Version verification (done):**
- `@openai/codex`: 0.118.0 — verified via `codex --version` on 2026-04-02
- `openai` npm: 6.33.0 — verified via `npm show openai version` on 2026-04-02
- `codex-plugin-cc`: 1.0.2 — verified from `~/.claude/plugins/installed_plugins.json` on 2026-04-02
- Node.js: v22.22.0 — verified via `node --version` on 2026-04-02

---

## Architecture Patterns

### Recommended Project Structure

```
~/.claude/hooks/
├── codex-router.js        # NEW: PreToolUse routing hook (opt-in, OFF by default)
└── codex-token-logger.js  # NEW: PostToolUse token logging after Codex calls
                           # (existing hooks unchanged)

~/.claude/settings.json    # USER SCOPE: add new hook entries here, not project scope
                           # (routing applies across all projects)

/home/alucard/projects/Claude_X_Codex/
├── AGENTS.md              # NEW: Codex project brief (D-01 through D-04)
└── .planning/
    ├── token-log.jsonl    # NEW: append-only token tracking
    └── codex-handoff/     # NEW: filesystem-based plan-to-Codex communication dir
        └── {id}.md        # handoff spec files (created by routing hook)
```

### Pattern 1: Node.js Hook Script (stdin/stdout JSON)

**What:** All hooks read JSON from stdin, process it, write JSON to stdout. Timeout guard on stdin prevents hangs. Exit 0 for success, exit 2 to block. This is the established GSD pattern.

**When to use:** Every new hook script in this phase.

**Example (from existing `gsd-context-monitor.js`):**
```javascript
// Source: ~/.claude/hooks/gsd-context-monitor.js (live reference implementation)
const stdinTimeout = setTimeout(() => process.exit(0), 10000);
process.stdin.setEncoding('utf8');
process.stdin.on('data', chunk => input += chunk);
process.stdin.on('end', () => {
  clearTimeout(stdinTimeout);
  try {
    const data = JSON.parse(input);
    // ... processing ...
    const output = {
      hookSpecificOutput: {
        hookEventName: 'PostToolUse',
        additionalContext: message  // injected into Claude's context
      }
    };
    process.stdout.write(JSON.stringify(output));
  } catch (e) {
    process.exit(0); // silent fail — never block tool execution
  }
});
```

### Pattern 2: PreToolUse Routing Hook

**What:** Intercepts Write/Edit tool calls during plan execution. Checks if routing is enabled via project config flag. If enabled and task matches routing criteria, injects `additionalContext` telling Opus to delegate to Codex instead.

**When to use:** FNDTN-03, ROUT-01. This is the primary routing enforcement point.

**Hook registration in `~/.claude/settings.json`:**
```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [
          {
            "type": "command",
            "command": "node \"/home/alucard/.claude/hooks/codex-router.js\"",
            "timeout": 5
          }
        ]
      }
    ]
  }
}
```

**Opt-in config check pattern:**
```javascript
// Source: pattern from gsd-context-monitor.js config check
const configPath = path.join(cwd, '.claude', 'settings.json');
if (fs.existsSync(configPath)) {
  const config = JSON.parse(fs.readFileSync(configPath, 'utf8'));
  if (!config.codex?.routing_enabled) {
    process.exit(0);  // routing OFF by default (D-07)
  }
}
```

**Project-level opt-in (`.claude/settings.json`):**
```json
{
  "codex": {
    "routing_enabled": true
  }
}
```

### Pattern 3: `codex exec --json` subprocess with timeout

**What:** Spawns `codex exec --json` as a child process. Streams JSONL output. Extracts `event_msg` with `token_count` info. Kills process at 300s. Always runs through this pattern — never interactive `codex` TUI.

**When to use:** FNDTN-02. Every Codex CLI invocation from a hook.

```javascript
// Source: timeout pattern from PITFALLS.md Pitfall 1, codex exec --help verified 2026-04-02
const { spawn } = require('child_process');

function runCodexExec(prompt, cwd, timeoutMs = 300000) {
  return new Promise((resolve, reject) => {
    const proc = spawn(
      'codex',
      ['exec', '--json', '--full-auto', '--model', 'gpt-5.4', prompt],
      { cwd, env: process.env }
    );

    const killTimer = setTimeout(() => {
      proc.kill('SIGTERM');
      reject(new Error('codex exec timed out after 300s'));
    }, timeoutMs);

    let output = '';
    proc.stdout.on('data', chunk => { output += chunk; });
    proc.on('close', (code) => {
      clearTimeout(killTimer);
      resolve({ code, output });
    });
    proc.on('error', (err) => {
      clearTimeout(killTimer);
      reject(err);
    });
  });
}
```

### Pattern 4: Token Log JSONL Append

**What:** After every Codex call, append one JSON line to `.planning/token-log.jsonl`. One record per call. Never overwrite. Format covers both CLI (from session JSONL) and API (from response.usage) calls.

**When to use:** TRCK-01, TRCK-02.

```javascript
// JSONL record schema (verified against live Codex session files 2026-04-02)
const record = {
  timestamp: new Date().toISOString(),
  model: 'gpt-5.4',               // or 'gpt-5.4-mini' for API calls
  source: 'cli',                   // 'cli' | 'api'
  task_type: 'implementation',     // 'implementation' | 'test-gen' | 'review' | 'bulk-ops'
  session_id: data.session_id,
  tokens: {
    input: totalTokenUsage.input_tokens,
    cached_input: totalTokenUsage.cached_input_tokens,
    output: totalTokenUsage.output_tokens,
    reasoning_output: totalTokenUsage.reasoning_output_tokens || 0
  },
  cost_usd: computeCost(totalTokenUsage, 'gpt-5.4'),  // computed, not provider-reported
  rate_limit_pct: rateLimits?.primary?.used_percent || null
};
fs.appendFileSync('.planning/token-log.jsonl', JSON.stringify(record) + '\n', 'utf8');
```

### Pattern 5: AGENTS.md Structure

**What:** File at repo root read by Codex CLI automatically before every task. Contains project brief, stack, conventions, guardrails, and the hard rule that Opus is sole architect.

**When to use:** FNDTN-01, ROUT-01.

```markdown
# AGENTS.md — Codex Project Brief

## What This Project Is
[Plain English description — matches D-03 communication style]

## Your Role
You are Codex (GPT-5.4), the executor model in a two-model setup.
Claude Opus 4.6 is the architect and orchestrator. You never make
architectural decisions. If a task requires a structural decision,
stop and say: "This requires an architectural decision. Please route
to Opus."

## What You Handle
- Clearly-defined implementation tasks (spec provided)
- Test generation
- Bulk file operations
- Code review and diff analysis

## What You Do NOT Handle
- Architecture decisions
- Debugging complex/unknown problems
- Exploratory analysis
- Anything not explicitly scoped in the handoff spec

## Tech Stack
[Current stack from CLAUDE.md]

## Conventions
[Populated at each GSD phase transition — D-04]

## Security Rules
- Never expose API keys
- Bind services to 127.0.0.1 only
- No sudo unless task explicitly requires it
```

### Anti-Patterns to Avoid

- **Calling raw `codex` CLI directly from hooks:** Always use `codex-companion.mjs` or `codex exec`. Raw CLI invocations bypass job state tracking and thread management.
- **`async: true` hooks for review tasks:** Async hooks cannot return `additionalContext` or block Claude's response. Review hooks that need to influence Claude's next action must be synchronous.
- **Fail-open routing:** If routing logic errors, the fallback must prompt the user — not auto-route to Opus. Auto-fallback to Opus doubles cost on every failed Codex attempt.
- **Storing API keys in hook script source:** Never reference `$OPENAI_API_KEY` inside hook script bodies. Keys must come from the process environment, never be logged or echoed.
- **Missing timeout wrapper:** Every `codex exec` call must have a 300s timeout. The background-process hang bug is open and unfixed in v0.118.0.

---

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Codex job state management | Custom job tracker | `codex-companion.mjs` from `codex-plugin-cc` v1.0.2 | Official bridge handles app-server protocol, thread IDs, background/foreground modes, status polling |
| Code review commands | Custom review prompt pipeline | `/codex:review`, `/codex:adversarial-review` slash commands | Already installed via `codex@openai-codex` v1.0.2 plugin |
| OpenAI API client | Raw `fetch()` calls | `openai@6.33.0` npm package | SDK handles retries, streaming, type safety, and auth header management |
| Token counting | Client-side tokenizer estimate | Provider-reported `usage` fields from API response / session JSONL | Provider counts are exact; client-side estimates drift by 5-15% |

**Key insight:** The `codex-companion.mjs` bridge is a full production implementation of the Codex App Server protocol. Any custom subprocess management or job tracking would need to re-implement its thread ID handling, turn capture state, and reconnection logic. Use it directly.

---

## Runtime State Inventory

Step 2.5: SKIPPED — this is a greenfield phase with no rename, refactor, or migration work. All artifacts (AGENTS.md, hook scripts, token-log.jsonl) are new files. No existing state is being renamed or migrated.

---

## Environment Availability Audit

| Dependency | Required By | Available | Version | Fallback |
|------------|------------|-----------|---------|----------|
| Node.js | All hook scripts | Yes | v22.22.0 | — |
| `codex` CLI | FNDTN-02, FNDTN-03, FNDTN-05 | Yes | 0.118.0 at `~/.npm-global/bin/codex` | — |
| `openai` npm | ROUT-04 (API fallback) | Yes | 6.33.0 (global) | — |
| `codex-plugin-cc` | `codex-companion.mjs` bridge | Yes | 1.0.2 | — |
| Claude Code hooks | FNDTN-02, FNDTN-03 | Yes | 2.1.90 (satisfies 2.0.65+) | — |
| `OPENAI_API_KEY` env var | FNDTN-05, API fallback | No | Not set | ChatGPT OAuth works for now, but FNDTN-05 requires API key mode |
| `CLAUDE_CODE_SUBPROCESS_ENV_SCRUB=1` | FNDTN-04 security | No | Not set | No — this is a required security control |

**Missing dependencies with no fallback:**
- `CLAUDE_CODE_SUBPROCESS_ENV_SCRUB=1` — must be added to `~/.bashrc` before any hooks that reference API keys are activated. This is a security requirement (FNDTN-04), not optional.

**Missing dependencies with fallback:**
- `OPENAI_API_KEY` — current ChatGPT OAuth auth works headlessly for now (stored refresh tokens in `~/.codex/auth.json`). However, FNDTN-05 explicitly requires API key mode for hook-invoked calls. The user must provide an OpenAI API key; it should be stored via `export OPENAI_API_KEY="..."` in `~/.bashrc` or a secrets manager — never in a plaintext file in the repo.

---

## Common Pitfalls

### Pitfall 1: Codex CLI Silent Background Hang

**What goes wrong:** `codex exec` enters an infinite "Waited for background terminal" loop when a spawned process exits via a non-standard path. The hook blocks forever; the entire Claude Code session hangs.

**Why it happens:** Known open bug in Codex CLI v0.111.0–v0.114.0+ (unresolved as of March 2026 in v0.118.0). The background terminal watcher fails to detect exit signals from certain subprocess exit methods.

**How to avoid:** Every `codex exec` call in a hook must use either a shell `timeout 300` prefix or a Node.js `setTimeout` kill on the child process at 300s. Use `--full-auto` or `--sandbox read-only` to prevent Codex from spawning persistent background processes.

**Warning signs:** Codex output shows "Waited for background terminal" repeated 3+ times. No new output for 60+ seconds. `ps aux | grep codex` still shows a process running past expected duration.

**Sources:** GitHub issues #14303, #14314, #13708

---

### Pitfall 2: API Key Exfiltration via Hook Scripts

**What goes wrong:** Hook scripts run with inherited environment variables. If `ANTHROPIC_BASE_URL` is overridden in a project `.claude/settings.json`, API keys can be exfiltrated before the user sees a trust prompt. Any hook that logs or echoes environment variables exposes keys in terminal history.

**Why it happens:** CVE-2025-59536, CVE-2026-21852 (patched in Claude Code 2.0.65+). Even with the patch, hooks running without `CLAUDE_CODE_SUBPROCESS_ENV_SCRUB=1` can access all environment variables.

**How to avoid:** (1) Confirm Claude Code version is 2.1.90 — already satisfies the requirement. (2) Set `CLAUDE_CODE_SUBPROCESS_ENV_SCRUB=1` in `~/.bashrc`. (3) Never `console.log(process.env)` or echo any env var in hook scripts. (4) Do not store `OPENAI_API_KEY` in any file committed to the repo.

**Warning signs:** Unexpected API spend spikes. Hook scripts that reference `process.env.OPENAI_API_KEY` directly in log statements.

---

### Pitfall 3: `codex exec --json` Token Field Source

**What goes wrong:** The `codex exec --json` JSONL stream emits `event_msg / token_count` events throughout execution. The first event often has `info: null` (before any tokens are used). If the token logger reads only the first event, it records zero tokens for the entire call.

**Why it happens:** Token count events are incremental. The `info` field is populated from the second turn onward. The relevant fields are inside `payload.info.total_token_usage`, not `payload.info.last_token_usage`.

**How to avoid:** The token logger must scan all JSONL lines from the `codex exec` output and take the last `token_count` event with a non-null `info` field. This gives cumulative totals for the full call.

**Verified field schema (from live `~/.codex/sessions/` file, 2026-04-02):**
```json
{
  "type": "event_msg",
  "payload": {
    "type": "token_count",
    "info": {
      "total_token_usage": {
        "input_tokens": 12251,
        "cached_input_tokens": 9728,
        "output_tokens": 242,
        "reasoning_output_tokens": 31,
        "total_tokens": 12493
      }
    },
    "rate_limits": {
      "primary": { "used_percent": 3.0 },
      "plan_type": "plus"
    }
  }
}
```

---

### Pitfall 4: ChatGPT Plus Rate Limit Quota

**What goes wrong:** The $20/mo Plus plan provides 30-150 Codex CLI requests per 5-hour rolling window, shared with ChatGPT web usage. A metering anomaly (March 2026) caused small tasks to consume 2% quota each. If the daily quota runs out, Codex CLI returns "usage limit reached" mid-session.

**How to avoid:** Reserve CLI quota for high-value, longer-running tasks. Route short advisory queries through the OpenAI API (`gpt-5.4-mini` via SDK) — these are not counted against CLI quota. The token logger should track `rate_limits.primary.used_percent` from each `token_count` event. If CLI quota exceeds 60%, switch to API mode automatically.

**Warning signs:** `codex exec` returns exit code non-zero with "usage limit reached" message. `rate_limits.primary.used_percent` in token log exceeds 80%.

---

### Pitfall 5: Routing Hook Blocking Write Operations

**What goes wrong:** A PreToolUse hook that takes 2+ seconds to run noticeably slows every Write/Edit call in the session, even when Codex routing is off. The existing `gsd-prompt-guard.js` exits in under 3ms for non-matching files. A routing hook that spawns a subprocess or does network I/O on every Write call will make the editor feel sluggish.

**How to avoid:** The routing hook must check the opt-in config flag first (2ms filesystem read) and exit immediately if routing is disabled. Only proceed to Codex invocation if routing is enabled AND the task matches routing criteria. Keep the hook's hot path (disabled or non-matching) under 5ms.

**Warning signs:** Write tool calls noticeably delayed in sessions where routing is not needed.

---

## Code Examples

Verified patterns from official sources:

### Hook output format — advisory context injection (PostToolUse)
```javascript
// Source: ~/.claude/hooks/gsd-context-monitor.js (live, verified 2026-04-02)
// Use this shape for non-blocking advisory messages:
const output = {
  hookSpecificOutput: {
    hookEventName: 'PostToolUse',
    additionalContext: 'message visible to Claude agent'
  }
};
process.stdout.write(JSON.stringify(output));
```

### Hook output format — deny/block (PreToolUse)
```javascript
// Source: ~/.claude/hooks/claude-settings-guard.js (live, verified 2026-04-02)
// Use this shape to block a tool call:
const output = {
  hookSpecificOutput: {
    hookEventName: 'PreToolUse',
    permissionDecision: 'deny',
    permissionDecisionReason: 'Reason shown to Claude when blocked'
  }
};
process.stdout.write(JSON.stringify(output));
process.exit(2);  // exit code 2 = deny
```

### Codex exec with --json and timeout
```javascript
// Source: codex exec --help (verified on v0.118.0, 2026-04-02)
// + timeout pattern from PITFALLS.md Pitfall 1
const { spawn } = require('child_process');
const proc = spawn('codex', ['exec', '--json', '--full-auto', prompt], { cwd });
const killTimer = setTimeout(() => proc.kill('SIGTERM'), 300000);  // 300s hard limit
proc.stdout.on('data', chunk => { /* accumulate JSONL */ });
proc.on('close', () => { clearTimeout(killTimer); /* process output */ });
```

### Token log record schema
```javascript
// Source: verified against live ~/.codex/sessions/2026/04/01/*.jsonl token_count events
// and TRCK-01/TRCK-02 requirements
const record = {
  timestamp: new Date().toISOString(),
  model: 'gpt-5.4',             // or 'gpt-5.4-mini'
  source: 'cli',                 // 'cli' | 'api'
  task_type: 'implementation',   // 'implementation'|'test-gen'|'review'|'bulk-ops'
  session_id: data.session_id,   // from hook stdin input
  tokens: {
    input: 12251,
    cached_input: 9728,
    output: 242,
    reasoning_output: 31
  },
  cost_usd: 0.0042,             // computed: apply per-provider pricing per token type
  rate_limit_pct: 3.0           // from rate_limits.primary.used_percent
};
fs.appendFileSync(tokenLogPath, JSON.stringify(record) + '\n', 'utf8');
```

### codex-companion.mjs task invocation
```javascript
// Source: ~/.claude/plugins/cache/openai-codex/codex/1.0.2/scripts/codex-companion.mjs
// Use codex-companion.mjs for structured job management (not raw codex CLI)
// Invoke as: node "$CLAUDE_PLUGIN_ROOT/scripts/codex-companion.mjs" task --background [prompt]
// Results retrievable via: node codex-companion.mjs result [job-id]
// Note: CLAUDE_PLUGIN_ROOT resolves to ~/.claude/plugins/cache/openai-codex/codex/1.0.2
```

---

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Manual copy-paste between Claude and Codex | Hook-based automatic routing | March 2026 (codex-plugin-cc v1.0.0) | Eliminates context-switching; routing happens at tool-call level |
| Codex OAuth browser login for every session | Stored OAuth refresh tokens (chatgpt mode) | CLI v0.100+ | Works headlessly after initial login; token refresh is automatic |
| Two separate `CLAUDE.md` files (one per tool) | One `AGENTS.md` at repo root (read by both) | Codex CLI v0.100+ | Single source of truth for project context across both models |

**Deprecated/outdated:**
- `codex login` browser popup: Still the default flow, but stored tokens in `~/.codex/auth.json` handle headless refresh automatically after initial login. For FNDTN-05, API key mode is more reliable and explicitly required.
- `commands/` subdirectory for GSD slash commands: Broke in Claude Code 2.1.89; migrated to `skills/` format. Any GSD plugin modifications must use the `skills/` extension pattern, not `commands/`.

---

## Open Questions

1. **Does `OPENAI_API_KEY` override chatgpt auth mode in Codex CLI?**
   - What we know: `codex login status` showed "Logged in using ChatGPT" even when `OPENAI_API_KEY=test` was set in env. The OAuth tokens take precedence.
   - What's unclear: Whether setting a real `OPENAI_API_KEY` causes Codex to use it over stored OAuth tokens, or whether `codex login` must be re-run with `--api-key` flag.
   - Recommendation: In Wave 0, test `OPENAI_API_KEY=sk-real-key codex exec --json "echo test"` to verify which auth mode is used. If OAuth still wins, the planner must include a `codex login` command that switches to API key mode.

2. **Opt-in config flag location**
   - What we know: D-07 requires routing to be OFF by default, enabled per project via `.claude/settings.json` or a config flag. The gsd-context-monitor.js pattern reads from `.planning/config.json`.
   - What's unclear: Whether the flag should live in `.claude/settings.json` (already read by Claude Code) or `.planning/config.json` (already read by GSD hooks) or a new `codex.json`.
   - Recommendation: Use `.claude/settings.json` project scope (`codex.routing_enabled: true`) to keep Codex config alongside Claude Code config. Consistent with existing settings architecture.

3. **Token cost calculation — GPT-5.4 pricing confirmation**
   - What we know: Research document lists $2.50/$15 per million (input/output) for GPT-5.4 standard. Pricing doubles above 272K input tokens.
   - What's unclear: Whether pricing has changed since April 2026 research cutoff.
   - Recommendation: Hard-code pricing constants in the token logger with a comment noting the verification date. Add a `WARN_STALE_PRICING` check if the log file is more than 30 days old.

---

## Project Constraints (from CLAUDE.md)

The following directives from `./CLAUDE.md` apply to this phase and constrain all implementation decisions:

| Directive | Impact on Phase 1 |
|-----------|------------------|
| **Budget**: $20/mo ChatGPT Plus; $15/day max API | Token logger must track daily spend with a $10/day ceiling (buffer before $15 cap) |
| **Security**: Never expose API keys in plaintext | `OPENAI_API_KEY` must be set via `~/.bashrc` export or secrets manager — not in any file in the repo |
| **Security**: Bind services to 127.0.0.1 | Not directly applicable to Phase 1 (no services started) |
| **Compatibility**: Must not break existing GSD/Superpowers workflows | New hooks added to `~/.claude/settings.json` must not interfere with existing `gsd-context-monitor.js`, `gsd-prompt-guard.js`, `claude-settings-guard.js` entries |
| **Runtime**: Codex CLI runs locally in terminal | All hook scripts invoke `codex exec` as a local subprocess — no cloud Codex or remote API for execution tasks |
| **Orchestration**: Opus always remains primary orchestrator | Routing hook output is `additionalContext` (advisory), not `permissionDecision: deny`. Opus decides whether to delegate; it is never blocked from proceeding |
| **Stack**: Node.js only — no Python, no Bash hook scripts | All new hooks are `.js` files following the existing pattern |
| **What NOT to use**: OpenAI Agents SDK, LangChain, Claude Flow, `opencode` CLI | Phase 1 uses only Node.js built-ins + `openai` SDK + `codex exec` subprocess |
| **GSD Workflow Enforcement** | This research was initiated via `/gsd:plan-phase` — compliant |

---

## Sources

### Primary (HIGH confidence)
- Claude Code hooks API documentation: https://code.claude.ai/docs/en/hooks (verified 2026-04-02)
- Codex CLI `--help` and `codex exec --help` output: verified on installed v0.118.0 (2026-04-02)
- `~/.claude/hooks/gsd-context-monitor.js` — live reference implementation for PostToolUse hook pattern
- `~/.claude/hooks/gsd-prompt-guard.js` — live reference implementation for PreToolUse hook pattern
- `~/.claude/hooks/claude-settings-guard.js` — live reference for `permissionDecision: deny` pattern
- `~/.claude/plugins/cache/openai-codex/codex/1.0.2/scripts/codex-companion.mjs` — official bridge implementation
- `~/.claude/plugins/cache/openai-codex/codex/1.0.2/scripts/lib/codex.mjs` — `getCodexLoginStatus`, token state management
- `~/.claude/plugins/cache/openai-codex/codex/1.0.2/scripts/lib/tracked-jobs.mjs` — job lifecycle and log patterns
- `~/.claude/plugins/cache/openai-codex/codex/1.0.2/hooks/hooks.json` — hook registration format for plugin hooks
- `~/.codex/sessions/2026/04/01/*.jsonl` — live Codex session files, verified `token_count` event schema (2026-04-02)
- `~/.codex/auth.json` — confirmed `auth_mode: chatgpt`, presence of OAuth refresh tokens (2026-04-02)
- `~/.claude/settings.json` — confirmed hook registration format and existing hook entries (2026-04-02)
- `.planning/research/PITFALLS.md` — project pitfall research backed by GitHub issues and CVE disclosures
- `.planning/research/SUMMARY.md` — project research summary with architecture patterns

### Secondary (MEDIUM confidence)
- `docs/research/opus-vs-codex-model-comparison.md` — 34-source model comparison, routing recommendations
- `codex-claude-code-power-user-research.md` — community patterns for Codex + Claude Code integration
- CVE-2025-59536 disclosure: https://research.checkpoint.com/2026/rce-and-api-token-exfiltration-through-claude-code-project-files-cve-2025-59536/
- Codex background hang: GitHub issues #14303, #14314, #13708

### Tertiary (context only)
- `CLAUDE.md` — technology stack decisions and "What NOT to Use" constraints
- `.planning/REQUIREMENTS.md` — full v1 requirements with phase traceability

---

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH — all versions verified from live system (2026-04-02)
- Architecture patterns: HIGH — derived from live hook source code, not just docs
- Token schema: HIGH — verified from live `~/.codex/sessions/` JSONL files
- Security requirements: HIGH — CVE references and live env var state confirmed
- Auth headless behavior: MEDIUM — OAuth refresh token behavior inferred from `auth.json` structure; API key override behavior untested

**Research date:** 2026-04-02
**Valid until:** 2026-05-02 (30 days for stable; re-verify Codex CLI version and pricing before then)
