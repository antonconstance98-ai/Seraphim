# Technology Stack

**Project:** Claude X Codex — Multi-Model AI Coding Agent Integration
**Researched:** 2026-04-02
**Overall Confidence:** HIGH (verified against live system, official docs, and actual installed versions)

---

## Recommended Stack

### Runtime: Node.js (Already Installed)

| Technology | Version | Purpose | Why |
|------------|---------|---------|-----|
| Node.js | v22.22.0 (installed) | Hook scripts, orchestration logic | All existing GSD/Superpowers hooks are Node.js scripts. Matching the runtime eliminates toolchain friction. |
| npm | 10.9.4 (installed) | Package management | Already present; no additional toolchain needed |

**Verified from:** `node --version` on target system — v22.22.0 confirmed.

### Integration Layer 1: Codex CLI (Already Installed)

| Technology | Version | Purpose | Why |
|------------|---------|---------|-----|
| `@openai/codex` CLI | 0.118.0 (installed at `~/.npm-global/bin/codex`) | Non-interactive Codex execution, code review | Installed and working. Uses ChatGPT Plus subscription (free under $20/mo plan). Avoids API billing for execution tasks. |

**Key Codex CLI invocation patterns (all verified from `codex --help` and live test):**

```bash
# Non-interactive execution with JSON output (for hook integration)
codex exec --json --ephemeral -m gpt-5.4 "implement X from this spec"

# Non-interactive code review on current git changes
codex review --uncommitted

# Review against a base branch
codex review --base main

# Full-auto mode (no approval prompts, sandboxed)
codex exec --full-auto "run tests and fix failures"

# Capture last response to file (for structured output)
codex exec -o /tmp/codex-output.txt "review this plan"

# Override model per call
codex exec -m gpt-5.4-mini --json "generate unit tests for auth.ts"

# Piped spec from stdin
echo "Implement input validation in auth.ts" | codex exec --json -
```

**JSONL output format (verified from live `codex exec --json` test):**
```jsonl
{"type":"thread.started","thread_id":"<uuid>"}
{"type":"turn.started"}
{"type":"item.completed","item":{"id":"item_0","type":"agent_message","text":"..."}}
{"type":"turn.completed","usage":{"input_tokens":23858,"cached_input_tokens":15232,"output_tokens":158}}
```

The `turn.completed` event contains the token usage needed for cost tracking.

**Confidence:** HIGH — verified from live execution on this system.

### Integration Layer 2: Claude Code Hooks Infrastructure

The hooks system is the primary mechanism for injecting Codex calls into Claude Code's workflow.

| Component | Version/Status | Purpose | Why |
|-----------|---------------|---------|-----|
| Claude Code hooks | Current (v2.1.89 session verified) | Trigger Codex at lifecycle events | Native to Claude Code; no third-party dependency; already used by GSD hooks |
| `~/.claude/settings.json` | User scope | Configure hooks globally | Persists across all projects; correct scope for cross-project Codex integration |
| `.claude/settings.json` | Project scope | Project-specific hook overrides | Per-project config; shareable via git |

**Hook event types relevant to this project (all verified from official docs):**

| Event | Fires When | Can Block? | Codex Use Case |
|-------|-----------|-----------|----------------|
| `PostToolUse` | After tool executes | No (advisory) | Trigger Codex review after Write/Edit/Bash |
| `Stop` | Before Claude finishes responding | Yes | Force cross-model review before sign-off |
| `SubagentStop` | Subagent completes | Yes | Validate subagent work via Codex |
| `TaskCompleted` | GSD task marked done | Yes | Gate completion on Codex review |
| `UserPromptSubmit` | User submits prompt | Yes | Inject Codex plan review context |
| `SessionStart` | Session begins | No | Load Codex routing config |

**Hook script pattern (matches existing GSD hooks):**

```javascript
#!/usr/bin/env node
// All hooks: read JSON from stdin, write JSON to stdout, exit 0 = success, exit 2 = block

let input = '';
const timeout = setTimeout(() => process.exit(0), 10000); // safety timeout
process.stdin.setEncoding('utf8');
process.stdin.on('data', chunk => input += chunk);
process.stdin.on('end', () => {
  clearTimeout(timeout);
  try {
    const data = JSON.parse(input);
    // data.tool_name, data.tool_input, data.tool_response, data.session_id, data.cwd
    
    // To inject context into Claude's next response:
    const output = {
      hookSpecificOutput: {
        hookEventName: "PostToolUse",  // Must match event type
        additionalContext: "Codex review: ..."
      }
    };
    process.stdout.write(JSON.stringify(output));
  } catch (e) {
    process.exit(0); // Silent fail — never block Claude
  }
});
```

**Hook output format — key fields:**
- `additionalContext` — text injected into Claude's context (max 10,000 chars)
- `decision: "block"` — blocks Claude's action (PostToolUse only)
- `reason` — message shown to Claude when blocked
- `continue: false` — stops all hook processing

**Confidence:** HIGH — verified from official docs at `code.claude.com/docs/en/hooks` and cross-checked against existing GSD hook implementations.

### Integration Layer 3: OpenAI Node.js SDK (for API calls)

Use only when Codex CLI overhead is impractical (model-to-model communication, quick structured queries).

| Technology | Version | Purpose | Why |
|------------|---------|---------|-----|
| `openai` npm package | 6.33.0 (latest as of 2026-04-02) | Direct API calls to GPT-5.4/GPT-5.4-mini | Lower overhead than spawning a CLI process when you need a 1-2 sentence structured response |

**Installation:**
```bash
npm install openai@6.33.0
```

**Minimal usage pattern for hooks:**
```javascript
const OpenAI = require('openai');
const client = new OpenAI({ apiKey: process.env.OPENAI_API_KEY });

const response = await client.chat.completions.create({
  model: 'gpt-5.4-mini',  // cheapest for background tasks
  messages: [{ role: 'user', content: prompt }],
  max_tokens: 500
});

// Token tracking — the usage field is always present in non-streaming responses
const tokens = response.usage;
// { prompt_tokens: N, completion_tokens: N, total_tokens: N }
```

**When to use API vs CLI:**
- API: structured JSON responses needed, sub-5-second latency required, no disk writes needed
- CLI: autonomous file editing needed, use ChatGPT Plus subscription (free), long-running tasks

**Confidence:** HIGH — verified npm version 6.33.0 from `npm show openai`.

### Integration Layer 4: Official Codex Plugin for Claude Code

Released March 30, 2026 by OpenAI. Wraps the local `codex` binary. Adds `/codex:*` commands to Claude Code.

| Component | Status | Commands | Why |
|-----------|--------|---------|-----|
| `codex-plugin-cc` | Available in Claude Code marketplace | `/codex:review`, `/codex:adversarial-review`, `/codex:rescue`, `/codex:status`, `/codex:result`, `/codex:cancel` | Official, maintained by OpenAI, uses existing auth/config/MCP — zero additional setup beyond install |

**Installation:**
```
/plugin marketplace add openai/codex-plugin-cc
/plugin install codex@openai-codex
/reload-plugins
/codex:setup
```

**Use this for:** Manual review triggers, adversarial review during planning phases, delegating investigation tasks. Do NOT use for automated hook-driven review (that requires direct `codex exec` calls).

**GitHub:** https://github.com/openai/codex-plugin-cc

**Confidence:** HIGH — verified from research doc and smartscope.blog analysis.

### Token Tracking: JSONL Session Files

| Mechanism | Location | Data Available |
|-----------|----------|---------------|
| Claude Code sessions | `~/.claude/projects/<hash>/<session-id>.jsonl` | `input_tokens`, `output_tokens`, `cache_creation_input_tokens`, `cache_read_input_tokens`, model name, timestamp |
| Codex CLI `--json` output | stdout JSONL | `input_tokens`, `cached_input_tokens`, `output_tokens` from `turn.completed` event |
| OpenAI API response | `response.usage` | `prompt_tokens`, `completion_tokens`, `total_tokens` |

**Verified Claude Code JSONL schema (from live session file):**
```json
{
  "type": "assistant",
  "message": {
    "model": "claude-opus-4-6",
    "usage": {
      "input_tokens": 2,
      "cache_creation_input_tokens": 40981,
      "cache_read_input_tokens": 0,
      "output_tokens": 30
    }
  },
  "timestamp": "2026-04-01T18:06:28.426Z",
  "sessionId": "a95cda63-...",
  "cwd": "/path/to/project"
}
```

**Token tracking storage:** Write to `/tmp/codex-tokens-{session_id}.json` (matches GSD's existing pattern for bridge files). Aggregate and report via `PostToolUse` hook or dedicated `/gsd:cost-report` command.

**Confidence:** HIGH — schema verified from live `~/.claude/projects/` JSONL files.

### Supporting Tools (Optional, Not Required for MVP)

| Tool | Version | Purpose | Install When |
|------|---------|---------|-------------|
| `ccusage` | latest | CLI analysis of Claude Code JSONL usage files | If reporting needs to extend beyond custom scripts |
| `spec2commit` | latest | Automated spec-to-commit workflow | If spec-driven loop becomes a recurring pattern |

---

## Alternatives Considered

| Category | Recommended | Alternative | Why Not |
|----------|-------------|-------------|---------|
| Hook language | Node.js (existing pattern) | Bash | GSD/Superpowers hooks already in Node.js; JSON parsing with jq is fragile; Node has better error handling |
| Hook language | Node.js (existing pattern) | Python | Would introduce a second runtime for no gain; `python3 -m json.tool` incurs startup overhead in hot paths |
| Codex invocation | `codex exec --json` subprocess | OpenAI API with assistants | CLI uses ChatGPT Plus subscription (free); API charges per token; assistants API has higher overhead |
| Model-to-model comms | OpenAI SDK direct call | Codex CLI | CLI startup ~1-2s is too slow for quick advisory context injection; API is faster for synchronous advisory |
| Config file location | `~/.claude/settings.json` | `.claude/settings.json` (project) | Codex routing applies to all projects, not just this one; user scope is correct |
| Token aggregation | Custom JSONL reader | LiteLLM proxy | LiteLLM requires a running service; overkill for single-user terminal workflow |

---

## What NOT to Use

| Technology | Reason |
|------------|--------|
| `OpenAI Agents SDK` | Adds a heavyweight framework dependency for what are simple subprocess calls. The hook + CLI pattern is sufficient and has zero framework lock-in. |
| `Claude Flow / Ruflo` | 60+ agents, 259 MCP tools — massive overkill for a plugin modification. Introduces opaque orchestration that makes debugging hard. |
| `opencode` CLI | Replaces both Claude Code and Codex CLI. We want to augment existing tools, not replace them. |
| `LangChain` / `LlamaIndex` | No need for a chain framework when the "chain" is two CLI tools passing files between them. Framework adds 50+ transitive dependencies. |
| GPT-5.4 Pro ($30/$180/Mtok) | Prohibitively expensive. GPT-5.4 standard provides equivalent quality for our use cases. Only needed for extended thinking mode, which this project doesn't require. |
| `Codex Cloud` | Requires paid ChatGPT Pro ($200/mo). User has Plus ($20/mo). Local Codex CLI is sufficient and free under subscription. |
| `async: true` hooks for review | Asynchronous hooks don't block Claude's response. Review needs to inject context synchronously to influence Claude's next action. Use sync hooks with `additionalContext`. |

---

## Hook Configuration Skeleton

This is the canonical structure for adding Codex hooks to `~/.claude/settings.json`:

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit|MultiEdit",
        "hooks": [
          {
            "type": "command",
            "command": "node \"/home/alucard/.claude/hooks/codex-review.js\"",
            "timeout": 60,
            "statusMessage": "Running Codex review..."
          }
        ]
      }
    ],
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "node \"/home/alucard/.claude/hooks/codex-stop-gate.js\"",
            "timeout": 90,
            "statusMessage": "Codex cross-model check..."
          }
        ]
      }
    ]
  }
}
```

**Note:** `Stop` hook has no `matcher` field (it fires on every stop event). `PostToolUse` uses matcher to target only file-writing tools.

---

## Installation Steps

```bash
# 1. Verify prerequisites (already satisfied)
node --version    # v22.22.0
codex --version   # codex-cli 0.118.0

# 2. Install OpenAI SDK (for API-based calls)
cd ~/.claude
npm install openai@6.33.0

# 3. Install official Codex plugin (inside Claude Code)
/plugin marketplace add openai/codex-plugin-cc
/plugin install codex@openai-codex
/reload-plugins

# 4. Ensure OPENAI_API_KEY is set for API fallback
# Add to ~/.bashrc or use a secret manager:
# export OPENAI_API_KEY="sk-..."

# 5. Verify Codex CLI auth
codex login  # or confirm existing auth via codex --version test
```

---

## Sources

- Official Claude Code Hooks API: https://code.claude.com/docs/en/hooks (verified 2026-04-02)
- Official Claude Code Cost Tracking: https://code.claude.com/docs/en/costs (verified 2026-04-02)
- Codex CLI `--help` output: verified on installed v0.118.0
- OpenAI Node.js SDK: https://github.com/openai/openai-node (version 6.33.0 verified via `npm show openai`)
- codex-plugin-cc: https://github.com/openai/codex-plugin-cc (verified from research doc + smartscope.blog)
- Live session JSONL schema: verified from `~/.claude/projects/` files on this system
- Codex exec JSONL event schema: verified from live `codex exec --json` test on v0.118.0
- ccusage tool: https://github.com/ryoppippi/ccusage
- Power user patterns research: `/home/alucard/projects/Claude_X_Codex/codex-claude-code-power-user-research.md`
