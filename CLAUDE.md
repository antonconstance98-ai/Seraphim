<!-- GSD:project-start source:PROJECT.md -->
## Project

**Claude X Codex**

A multi-model integration that adds OpenAI Codex (GPT-5.4) capabilities into an existing Claude Code workflow. It modifies the Claude Code configuration (hooks, agents), the GSD (Get Shit Done) plugin, and the Superpowers plugin so that Claude Opus 4.6 acts as the orchestrator/architect while Codex handles implementation execution — with a cross-model plan review loop before any code is written. The goal is better results at lower cost by routing each task to the model that's best at it.

**Core Value:** Every task goes to the model that's best at it — Opus for reasoning and architecture, Codex for fast execution — with cross-model review catching what either model misses alone.

### Constraints

- **Budget**: $20/mo ChatGPT Plus subscription; $15/day max API spend; prefer CLI over API billing
- **Security**: Never expose API keys in plaintext; use environment variables; bind services to 127.0.0.1
- **Compatibility**: Must work with existing GSD and Superpowers plugin versions without breaking current workflows
- **Runtime**: Codex CLI runs locally in terminal; API calls use OpenAI SDK
- **Orchestration**: Opus always remains the primary orchestrator; Codex never makes architectural decisions autonomously
<!-- GSD:project-end -->

<!-- GSD:stack-start source:research/STACK.md -->
## Technology Stack

## Recommended Stack
### Runtime: Node.js (Already Installed)
| Technology | Version | Purpose | Why |
|------------|---------|---------|-----|
| Node.js | v22.22.0 (installed) | Hook scripts, orchestration logic | All existing GSD/Superpowers hooks are Node.js scripts. Matching the runtime eliminates toolchain friction. |
| npm | 10.9.4 (installed) | Package management | Already present; no additional toolchain needed |
### Integration Layer 1: Codex CLI (Already Installed)
| Technology | Version | Purpose | Why |
|------------|---------|---------|-----|
| `@openai/codex` CLI | 0.118.0 (installed at `~/.npm-global/bin/codex`) | Non-interactive Codex execution, code review | Installed and working. Uses ChatGPT Plus subscription (free under $20/mo plan). Avoids API billing for execution tasks. |
# Non-interactive execution with JSON output (for hook integration)
# Non-interactive code review on current git changes
# Review against a base branch
# Full-auto mode (no approval prompts, sandboxed)
# Capture last response to file (for structured output)
# Override model per call
# Piped spec from stdin
### Integration Layer 2: Claude Code Hooks Infrastructure
| Component | Version/Status | Purpose | Why |
|-----------|---------------|---------|-----|
| Claude Code hooks | Current (v2.1.89 session verified) | Trigger Codex at lifecycle events | Native to Claude Code; no third-party dependency; already used by GSD hooks |
| `~/.claude/settings.json` | User scope | Configure hooks globally | Persists across all projects; correct scope for cross-project Codex integration |
| `.claude/settings.json` | Project scope | Project-specific hook overrides | Per-project config; shareable via git |
| Event | Fires When | Can Block? | Codex Use Case |
|-------|-----------|-----------|----------------|
| `PostToolUse` | After tool executes | No (advisory) | Trigger Codex review after Write/Edit/Bash |
| `Stop` | Before Claude finishes responding | Yes | Force cross-model review before sign-off |
| `SubagentStop` | Subagent completes | Yes | Validate subagent work via Codex |
| `TaskCompleted` | GSD task marked done | Yes | Gate completion on Codex review |
| `UserPromptSubmit` | User submits prompt | Yes | Inject Codex plan review context |
| `SessionStart` | Session begins | No | Load Codex routing config |
#!/usr/bin/env node
- `additionalContext` — text injected into Claude's context (max 10,000 chars)
- `decision: "block"` — blocks Claude's action (PostToolUse only)
- `reason` — message shown to Claude when blocked
- `continue: false` — stops all hook processing
### Integration Layer 3: OpenAI Node.js SDK (for API calls)
| Technology | Version | Purpose | Why |
|------------|---------|---------|-----|
| `openai` npm package | 6.33.0 (latest as of 2026-04-02) | Direct API calls to GPT-5.4/GPT-5.4-mini | Lower overhead than spawning a CLI process when you need a 1-2 sentence structured response |
- API: structured JSON responses needed, sub-5-second latency required, no disk writes needed
- CLI: autonomous file editing needed, use ChatGPT Plus subscription (free), long-running tasks
### Integration Layer 4: Official Codex Plugin for Claude Code
| Component | Status | Commands | Why |
|-----------|--------|---------|-----|
| `codex-plugin-cc` | Available in Claude Code marketplace | `/codex:review`, `/codex:adversarial-review`, `/codex:rescue`, `/codex:status`, `/codex:result`, `/codex:cancel` | Official, maintained by OpenAI, uses existing auth/config/MCP — zero additional setup beyond install |
### Token Tracking: JSONL Session Files
| Mechanism | Location | Data Available |
|-----------|----------|---------------|
| Claude Code sessions | `~/.claude/projects/<hash>/<session-id>.jsonl` | `input_tokens`, `output_tokens`, `cache_creation_input_tokens`, `cache_read_input_tokens`, model name, timestamp |
| Codex CLI `--json` output | stdout JSONL | `input_tokens`, `cached_input_tokens`, `output_tokens` from `turn.completed` event |
| OpenAI API response | `response.usage` | `prompt_tokens`, `completion_tokens`, `total_tokens` |
### Supporting Tools (Optional, Not Required for MVP)
| Tool | Version | Purpose | Install When |
|------|---------|---------|-------------|
| `ccusage` | latest | CLI analysis of Claude Code JSONL usage files | If reporting needs to extend beyond custom scripts |
| `spec2commit` | latest | Automated spec-to-commit workflow | If spec-driven loop becomes a recurring pattern |
## Alternatives Considered
| Category | Recommended | Alternative | Why Not |
|----------|-------------|-------------|---------|
| Hook language | Node.js (existing pattern) | Bash | GSD/Superpowers hooks already in Node.js; JSON parsing with jq is fragile; Node has better error handling |
| Hook language | Node.js (existing pattern) | Python | Would introduce a second runtime for no gain; `python3 -m json.tool` incurs startup overhead in hot paths |
| Codex invocation | `codex exec --json` subprocess | OpenAI API with assistants | CLI uses ChatGPT Plus subscription (free); API charges per token; assistants API has higher overhead |
| Model-to-model comms | OpenAI SDK direct call | Codex CLI | CLI startup ~1-2s is too slow for quick advisory context injection; API is faster for synchronous advisory |
| Config file location | `~/.claude/settings.json` | `.claude/settings.json` (project) | Codex routing applies to all projects, not just this one; user scope is correct |
| Token aggregation | Custom JSONL reader | LiteLLM proxy | LiteLLM requires a running service; overkill for single-user terminal workflow |
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
## Hook Configuration Skeleton
## Installation Steps
# 1. Verify prerequisites (already satisfied)
# 2. Install OpenAI SDK (for API-based calls)
# 3. Install official Codex plugin (inside Claude Code)
# 4. Ensure OPENAI_API_KEY is set for API fallback
# Add to ~/.bashrc or use a secret manager:
# export OPENAI_API_KEY="sk-..."
# 5. Verify Codex CLI auth
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
<!-- GSD:stack-end -->

<!-- GSD:conventions-start source:CONVENTIONS.md -->
## Conventions

Conventions not yet established. Will populate as patterns emerge during development.
<!-- GSD:conventions-end -->

<!-- GSD:architecture-start source:ARCHITECTURE.md -->
## Architecture

Architecture not yet mapped. Follow existing patterns found in the codebase.
<!-- GSD:architecture-end -->

<!-- GSD:workflow-start source:GSD defaults -->
## GSD Workflow Enforcement

Before using Edit, Write, or other file-changing tools, start work through a GSD command so planning artifacts and execution context stay in sync.

Use these entry points:
- `/gsd:quick` for small fixes, doc updates, and ad-hoc tasks
- `/gsd:debug` for investigation and bug fixing
- `/gsd:execute-phase` for planned phase work

Do not make direct repo edits outside a GSD workflow unless the user explicitly asks to bypass it.
<!-- GSD:workflow-end -->



<!-- GSD:profile-start -->
## Developer Profile

> Profile not yet configured. Run `/gsd:profile-user` to generate your developer profile.
> This section is managed by `generate-claude-profile` -- do not edit manually.
<!-- GSD:profile-end -->
