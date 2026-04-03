# Phase 13: Codex Execution Pipeline - Research

**Researched:** 2026-04-03
**Domain:** GSD executor transformation — thin orchestrator pattern with Codex CLI + MiniMax fallback
**Confidence:** HIGH

---

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions

- **D-01:** Handoff spec format: natural language task description + file paths. Tells Codex/MiniMax WHAT to do in plain English.
- **D-02:** Each handoff spec includes: task description, target file path(s), action (create/modify/delete), specification of changes, verification command.
- **D-03:** MiniMax fallback: handoff spec also includes current file content. MiniMax returns full modified content as text; orchestrator writes it to disk.
- **D-04:** gsd-executor thin orchestrator runs on **Sonnet 4.6 ($3/$15 per Mtok)**. Only reads plans, generates handoff specs, validates outputs, manages git commits.
- **D-05:** Set `executor_model` to `sonnet` in GSD model profile configuration (GSD config change, not a hook change).
- **D-06:** Per-task execution flow: (1) read task from PLAN.md, (2) generate natural language handoff spec, (3) invoke `codex exec --full-auto --json [spec]`, (4) validate output (check git diff, run verification command), (5) commit atomically per task, (6) write SUMMARY.md after all tasks.
- **D-07:** Preserves ALL existing GSD protocols: atomic per-task commits, deviation handling, STATE.md updates, SUMMARY.md creation, checkpoint pausing.
- **D-08:** Fallback chain: Codex CLI (free via subscription) → MiniMax API ($0.30/$1.20) → prompt user (fail-closed).
- **D-09:** Rate-limit detection via `runWithFallback()` from Phase 8: exit codes, stderr "rate limit"/"quota"/"usage limit", HTTP 429 in JSONL, timeout with no output, `rate_limit_pct >= 95`.
- **D-10:** MiniMax fallback: orchestrator reads target file(s), includes content in handoff spec, sends to MiniMax API, receives modified content as text, writes to disk using its own Write/Edit tools.
- **D-11:** Both Codex and MiniMax fail: prompt user with both error messages. Options: (1) wait and retry Codex, (2) check MINIMAX_API_KEY, (3) skip this task. Fail-closed — never silently fall back to Opus for code writing.
- **D-12:** Log every fallback event to `token-log.jsonl` with `source: 'cli-fallback'` or `source: 'api-fallback'`.

### Claude's Discretion

- Handoff spec prompt template wording
- Validation depth (basic grep vs running tests)
- How to handle multi-file tasks (one Codex call per file or one call for all files)
- Whether to batch small tasks into a single Codex invocation for efficiency

### Deferred Ideas (OUT OF SCOPE)

- Parallel Codex invocations for independent tasks within a plan (currently sequential, parallel later if bottleneck)
- Smart handoff spec complexity detection (auto-switch between simple and detailed specs based on task difficulty)
- Codex warm-up (pre-spawn Codex process to reduce cold-start latency)
</user_constraints>

---

## Summary

Phase 13 transforms the `gsd-executor.md` agent from a model that writes code directly into a thin orchestrator. Instead of writing code itself, the Sonnet-powered executor reads tasks from PLAN.md, generates natural language handoff specs, invokes Codex CLI (`codex exec --full-auto --json`), validates the output, and commits atomically. The MiniMax API serves as a rate-limit fallback — the orchestrator reads file contents, sends them with the spec, receives modified text, and writes files itself (since MiniMax has no filesystem access).

The core infrastructure is already built. `runCodexExec()` in `codex-exec.js` and `runWithFallback()` / `isCodexRateLimited()` in `minimax-exec.js` are fully implemented and verified. `gsd-executor.md` contains the complete GSD execution protocol (deviation rules, checkpoint handling, SUMMARY.md creation, STATE.md updates) and needs modification rather than replacement — the protocol sections stay intact, only the task execution step changes.

The `executor_model` already resolves to `sonnet` under the `balanced` profile without any config change. The primary work is: (1) modifying `gsd-executor.md` to insert the Codex handoff spec pattern into its task execution step, and (2) adding fallback event logging to `token-log.jsonl` per D-12.

**Primary recommendation:** Modify the `execute_tasks` step in `gsd-executor.md` to call `runCodexExec()` via a short helper embedded in the agent's Bash blocks, with MiniMax fallback via `runMinimax()` when `isCodexRateLimited()` returns true. All existing deviation rules, checkpoint protocols, and commit patterns remain unchanged.

---

## Standard Stack

### Core
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| `codex` CLI | 0.118.0 (installed at `~/.npm-global/bin/codex`) | Non-interactive code execution | Free via ChatGPT Plus subscription; sandboxed filesystem access |
| `minimax-exec.js` | 1.1.0 (Phase 8 output) | `runWithFallback()`, `runMinimax()`, `isCodexRateLimited()` | Already implements complete fallback chain with all D-09 detection methods |
| `codex-exec.js` | 1.1.0 (existing) | `runCodexExec()` for Codex CLI subprocess | Already implements 300s timeout, SIGTERM/SIGKILL, JSONL token parsing |
| `gsd-executor.md` | current | Agent definition to modify | Contains complete GSD execution protocol — modify, do not replace |
| `gsd-tools.cjs` | current | `commit`, `state`, `roadmap` commands | All existing commit/state/summary protocols use this |

### Supporting
| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| `codex-pricing.js` | current | `computeCodexCostStrict()` for MiniMax fallback cost logging | When logging fallback events to token-log.jsonl (D-12) |
| `codex-token-logger.js` | current | Existing token logging via `[CODEX_RESULT]` marker | Token logger reads this marker from tool output; fallback logging must use same format |
| Node.js | v22.22.0 | Hook script runtime | All existing hooks are Node.js; matches installed runtime |

**Environment Availability:**

| Dependency | Required By | Available | Version | Fallback |
|------------|------------|-----------|---------|----------|
| `codex` CLI | D-06 primary executor | Yes | 0.118.0 | MiniMax API (D-08) |
| `minimax-exec.js` | D-08/D-09 fallback | Yes | Phase 8 output | User prompt (D-11) |
| `codex-exec.js` | Codex subprocess wrapper | Yes | Phase 1 output | N/A (dependency of minimax-exec.js) |
| `MINIMAX_API_KEY` | MiniMax fallback | **Not set in current env** | N/A | User prompt (D-11) — acceptable per design |
| Node.js v22 | Hook scripts | Yes | v22.22.0 | N/A |

**MINIMAX_API_KEY note:** Not currently set in the shell environment. This is expected — MiniMax is the fallback, not the primary. When Codex is not rate-limited (the common case), the key is never needed. When Codex is rate-limited and the key is absent, `runMinimax()` returns `{ success: false, error: 'MINIMAX_API_KEY is not set' }`, which triggers D-11 (user prompt). This is correct behavior per the design.

---

## Architecture Patterns

### What Changes and What Stays Identical

The `gsd-executor.md` agent has two conceptual halves:

1. **Protocol layer** (unchanged): Load plan, deviation rules, checkpoint handling, TDD execution, task commit protocol, SUMMARY creation, self-check, STATE updates, ROADMAP updates, final commit. These ~400 lines stay unchanged.

2. **Task execution step** (modified): The `<step name="execute_tasks">` section currently says "Execute task" and writes code directly. Phase 13 replaces this single step with the handoff spec → Codex → validate → commit pattern.

### Pattern 1: Handoff Spec Generation

The orchestrator (Sonnet) generates a natural language spec from the task description in PLAN.md. The spec is assembled inline in the agent's reasoning, not via a separate file.

**Spec format (D-02):**
```
Task: [action] [target file(s)]
Action: create | modify | delete
File paths: [relative to project root]
Specification: [what the file should contain or how it should change]
Current content: [included only for MiniMax fallback — full file content]
Verification: [command to run after Codex completes, e.g., grep 'functionName' path/to/file.js]
```

**Template wording guidance (Claude's discretion):**
- Lead with the action verb: "Create", "Modify", "Add", "Delete"
- State the file path explicitly (Codex needs it to find/create the file)
- Include the full function signature or key identifiers for modify tasks
- Keep specs under ~500 words — Codex is better at focused specs than kitchen-sink prompts
- Include the verification command so Codex knows the acceptance criteria

### Pattern 2: Codex CLI Invocation

`runCodexExec()` already exists and handles all subprocess concerns. The executor agent invokes it via Bash to avoid loading Node modules inside the Claude context window:

```bash
# Inside gsd-executor.md task execution step
CODEX_RESULT=$(node -e "
const { runCodexExec } = require('/home/alucard/.claude/hooks/codex-exec.js');
const spec = $(cat /tmp/handoff-spec-${TASK_ID}.txt | node -e 'process.stdout.write(JSON.stringify(require(\"fs\").readFileSync(\"/dev/stdin\",\"utf8\")))');
runCodexExec(spec, { cwd: process.cwd(), timeoutMs: 300000 }).then(r => {
  process.stdout.write(JSON.stringify(r));
});
")
```

**Simpler pattern (recommended):** Since the agent can write a small Node.js script file, write the spec to a temp file and execute it:

```bash
# Write handoff spec to temp file
cat > /tmp/phase13-handoff.txt << 'SPEC_EOF'
[handoff spec content]
SPEC_EOF

# Invoke Codex CLI directly (matches runCodexExec() internals)
codex exec --full-auto --json --model gpt-5.4 "$(cat /tmp/phase13-handoff.txt)" 2>/tmp/phase13-stderr.txt
CODEX_EXIT=$?
CODEX_OUTPUT=$(cat /tmp/phase13-stdout.txt 2>/dev/null || true)
```

However, using `runCodexExec()` directly is preferred for consistency — it handles the SIGTERM/SIGKILL timeout, parses JSONL tokens, and returns a structured result the executor can log.

**Recommended invocation pattern — inline Node.js script:**

```javascript
// /tmp/phase13-exec.js (written by the executor, executed via Bash)
const { runCodexExec } = require('/home/alucard/.claude/hooks/codex-exec.js');
const spec = process.argv[2]; // passed as CLI arg to avoid shell escaping issues
runCodexExec(spec, { cwd: process.cwd(), timeoutMs: 300000 }).then(result => {
  console.log(JSON.stringify(result));
}).catch(err => {
  console.log(JSON.stringify({ success: false, error: err.message }));
});
```

```bash
CODEX_JSON=$(node /tmp/phase13-exec.js "$HANDOFF_SPEC")
CODEX_SUCCESS=$(echo "$CODEX_JSON" | node -e "const d=JSON.parse(require('fs').readFileSync('/dev/stdin','utf8')); process.stdout.write(String(d.success));")
```

### Pattern 3: Rate-Limit Detection and MiniMax Fallback

`isCodexRateLimited()` covers all D-09 detection methods. The executor calls it after `runCodexExec()` fails:

```javascript
// /tmp/phase13-fallback.js
const { runWithFallback, isCodexRateLimited } = require('/home/alucard/.claude/hooks/minimax-exec.js');
const spec = process.argv[2];
const fileContent = process.argv[3] || ''; // empty if file doesn't exist yet

// For MiniMax fallback, include current file content in spec (D-03/D-10)
const fullSpec = fileContent ? spec + '\n\nCurrent file content:\n```\n' + fileContent + '\n```' : spec;

runWithFallback(fullSpec, { taskCategory: 'execution', timeoutMs: 90000 }).then(result => {
  console.log(JSON.stringify(result));
});
```

**Critical distinction from runWithFallback() semantics:**
The existing `runWithFallback()` in `minimax-exec.js` is designed for review tasks — it passes the same prompt to both Codex and MiniMax. For execution tasks, the MiniMax path needs file content included in the spec (D-03), which the Codex path does NOT need (Codex has filesystem access). The executor must handle this asymmetry.

**Recommended approach:** Call `runCodexExec()` directly first. On failure, call `isCodexRateLimited()`, then call `runMinimax()` directly with the augmented spec (including file content). Do NOT use `runWithFallback()` for execution tasks — it passes the identical prompt to both models, which doesn't work for the MiniMax file-content-injection requirement.

This matches the Pattern established in Phase 9 (D-09 in Phase 9 context): "Direct runCodexExec (not runWithFallback) for Codex leg — prevents double-MiniMax spend".

### Pattern 4: MiniMax Output Handling (API-only model)

MiniMax has no filesystem access. It returns modified file content as text. The executor reads this text and writes it to disk:

```javascript
// After successful runMinimax() call:
const minimaxText = minimaxResult.text;
// Strip <think> wrapper if present (added by runMinimax when reasoningSplit:true)
const thinkEnd = minimaxText.indexOf('</think>');
const fileContent = thinkEnd !== -1 ? minimaxText.slice(thinkEnd + '</think>'.length).trim() : minimaxText;
// Write to disk via Write tool (executor has Write tool access per gsd-executor.md tools list)
```

**Extraction note:** The executor agent uses its own Write/Edit tools to write MiniMax output to disk — not another subprocess. This is the cleanest pattern since `gsd-executor.md` has `tools: Read, Write, Edit, Bash, Grep, Glob`.

### Pattern 5: Validation After Codex/MiniMax

The verification command from D-02 runs after the executor completes. For most tasks, this is a `grep` for a function name or a `node -e "require('./path')"` smoke test:

```bash
# Basic grep verification (Claude's discretion — suitable for most tasks)
grep -l "functionName" path/to/file.js && echo "VERIFIED" || echo "VERIFICATION FAILED"

# Module smoke test
node -e "const m = require('./path/to/module'); console.log('OK');" 2>&1
```

Validation depth (Claude's discretion): start with `git diff --stat` to confirm files changed, plus the grep verification command from the handoff spec. Tests are a second-pass option if the task explicitly requires them.

### Pattern 6: Fallback Event Logging to token-log.jsonl (D-12)

The token logger (`codex-token-logger.js`) reads `[CODEX_RESULT]` markers from tool output. For fallback events, the executor should emit the same marker format so the existing logger captures them:

```
[CODEX_RESULT] {"model":"minimax-m2.7","source":"api-fallback","task_type":"execution","tokens":{...},"rate_limit_pct":null}
```

The `source` field is what D-12 specifies: `'cli-fallback'` or `'api-fallback'`. The token logger already records `source` as a field — no schema changes needed.

**For Codex CLI success (source: 'codex-cli'):** The existing `runCodexExec()` → `parseCodexTokens()` path already emits this. No change needed for the happy path.

**For MiniMax fallback (source: 'api-fallback'):** The executor must emit the `[CODEX_RESULT]` marker after writing files to disk. The `runMinimax()` result includes `tokens` and `cost` — format these into the marker.

### Pattern 7: D-05 — executor_model Config Change

**Finding: No config change is needed.** The `balanced` model profile already maps `gsd-executor` to `sonnet` (verified: `gsd-tools.cjs init execute-phase 13` returns `executor_model: "sonnet"`). D-05 is already satisfied by the existing model-profiles.md configuration.

If the user is running a non-`balanced` profile, they can set:
```json
{
  "model_overrides": {
    "gsd-executor": "sonnet"
  }
}
```
in `.planning/config.json`. But for this project with `model_profile: balanced`, no action is needed.

### Anti-Patterns to Avoid

- **Using `runWithFallback()` for execution tasks:** It sends the same prompt to both models, but MiniMax needs file content injected (D-03/D-10) that Codex does not need. Use `runCodexExec()` + `isCodexRateLimited()` + `runMinimax()` separately.
- **Writing MiniMax output to disk via subprocess:** The executor agent has `Write` tool access. Use the Write tool directly, not a Bash subshell writing files.
- **Stripping `<think>` tags from MiniMax response before writing:** Strip them only when extracting file content. Log or discard reasoning but never write it to the target file.
- **Modifying the GSD protocol sections of gsd-executor.md:** Deviation rules, checkpoint protocol, SUMMARY creation, STATE updates — these must remain unchanged (D-07).
- **Blocking on Codex for non-rate-limit failures:** If Codex fails for reasons other than rate limiting (auth error, malformed spec, spawn failure), `isCodexRateLimited()` returns `false`. Do NOT attempt MiniMax fallback for these — they indicate a configuration problem. Route to D-11 user prompt instead.
- **Silently falling back to Opus:** D-11 explicitly forbids this. Fail-closed means prompt the user, not silently use a more expensive model.

---

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Codex subprocess with timeout | Custom spawn logic | `runCodexExec()` from `codex-exec.js` | Already handles 300s timeout, SIGTERM/SIGKILL, stdout/stderr collection, JSONL parsing |
| Rate-limit detection | Custom string matching | `isCodexRateLimited()` from `minimax-exec.js` | Covers all 4 D-09 detection methods including HTTP 429 in JSONL and rate_limit_pct threshold |
| MiniMax API calls | Raw OpenAI SDK client | `runMinimax()` from `minimax-exec.js` | Handles retry with backoff, AbortController timeout, reasoning extraction, token normalization, cost computation |
| Token/cost logging | Custom JSONL writer | `[CODEX_RESULT]` marker pattern + existing `codex-token-logger.js` | Logger already reads this marker; no schema changes needed |
| Codex JSONL output parsing | Manual line-by-line parsing | `parseCodexTokens()` from `codex-exec.js` | Handles both `turn.completed` format (v0.118.0+) and `event_msg` legacy format |

**Key insight:** The entire fallback chain infrastructure is already implemented in Phase 8. Phase 13's value is in wiring it into the executor agent definition, not rebuilding any of these primitives.

---

## Common Pitfalls

### Pitfall 1: Using runWithFallback() for Execution Tasks
**What goes wrong:** `runWithFallback()` sends the same prompt string to both Codex and MiniMax. MiniMax needs the current file content embedded in the prompt (D-03/D-10) because it has no filesystem access. Codex does NOT need file content (it reads files itself). Using the same prompt means either MiniMax gets no file content (wrong output) or Codex gets unnecessary file content (wasted tokens).
**Why it happens:** `runWithFallback()` was designed for review tasks where both models receive identical context.
**How to avoid:** Call `runCodexExec()` for the Codex attempt. If `isCodexRateLimited()` is true, read the target file(s) and call `runMinimax()` with the augmented spec. Keep the two code paths separate.

### Pitfall 2: Multi-File Tasks — One Call or Many?
**What goes wrong:** Sending all file changes in one Codex invocation can produce partial results — Codex modifies some files but not others, or makes inconsistent changes across files.
**Why it happens:** Codex is optimized for focused single-task specs. Multi-file specs increase ambiguity.
**How to avoid:** For multi-file tasks (Claude's discretion), split into one Codex invocation per file. This is sequential, not parallel (deferred), but ensures each file change is independently validated. Only batch files that are truly inseparable (e.g., a module and its index re-export).
**Warning signs:** Task spec includes 3+ files with distinct changes. Codex output touches wrong files or partially implements changes.

### Pitfall 3: MiniMax Returning Code With `<think>` Prefix
**What goes wrong:** `runMinimax()` prepends `<think>...\n</think>\n\n` to the text when reasoning is available. If the executor writes this raw text to a file, the `<think>` block appears at the top of the file.
**Why it happens:** `runMinimax()` preserves reasoning content for transparency. Callers must strip it.
**How to avoid:** After a successful `runMinimax()` call, find the `</think>` tag and take the content after it as the actual file content. If no `</think>` tag is present, the full text is the file content.

### Pitfall 4: Codex Non-Rate-Limit Failures Triggering MiniMax
**What goes wrong:** Codex fails due to a config problem (auth, bad spec, spawn error) and the executor tries MiniMax. MiniMax produces different output. The user sees confusing results and the root cause (Codex config) is hidden.
**Why it happens:** Code checks `success === false` and immediately falls back, without checking `isCodexRateLimited()`.
**How to avoid:** Always check `isCodexRateLimited(codexResult)`. Only attempt MiniMax if it returns `true`. For other failures, go directly to D-11 user prompt.

### Pitfall 5: Breaking the `[CODEX_RESULT]` Token Logging Chain
**What goes wrong:** The executor completes a task but the token logger doesn't record it. The dashboard shows zero cost for Phase 13 work.
**Why it happens:** The executor outputs the `[CODEX_RESULT]` marker in a Bash command result, but the tool_result the logger receives doesn't include it (e.g., it was written to stderr or a temp file instead of stdout).
**How to avoid:** Emit `[CODEX_RESULT] {...}` as the final line of stdout from the Bash block that invokes Codex/MiniMax. The logger reads `data.tool_result` from the PostToolUse hook — this maps to the Claude tool result, which is the content returned from the Bash tool call.

### Pitfall 6: MINIMAX_API_KEY Not Set in Agent Subprocess Context
**What goes wrong:** `MINIMAX_API_KEY` is set in the user's shell but not propagated to the Claude Code subprocess that runs the agent.
**Why it happens:** Claude Code hooks inherit environment differently from interactive shell sessions.
**How to avoid:** `runMinimax()` already handles this gracefully — it returns `{ success: false, error: 'MINIMAX_API_KEY is not set' }` which triggers the D-11 user prompt. The plan should include a verification step that confirms `MINIMAX_API_KEY` is accessible from within a Claude Code hook subprocess.

---

## Code Examples

### Verified Pattern: runCodexExec() Result Object Shape
Source: `~/.claude/hooks/codex-exec.js` (verified on disk)
```javascript
// Successful result:
{
  success: true,
  exitCode: 0,
  output: '...JSONL stdout...',  // raw JSONL from codex exec --json
  tokens: {
    input_tokens: 1234,
    cached_input_tokens: 0,
    output_tokens: 456,
    reasoning_output_tokens: 0,
    total_tokens: 1690,
    rate_limit_pct: null  // null in turn.completed format; number in event_msg format
  },
  rateLimitPct: null,
  error: null
}

// Failed result:
{
  success: false,
  exitCode: 1,  // or null for spawn errors
  output: '',
  tokens: null,
  rateLimitPct: null,
  error: 'codex exec exited with code 1'  // or timeout message, or spawn error
}
```

### Verified Pattern: isCodexRateLimited() Detection
Source: `~/.claude/hooks/minimax-exec.js` (verified on disk)
```javascript
function isCodexRateLimited(codexResult) {
  // 1. rate_limit_pct >= 95
  if (codexResult.rateLimitPct !== null && codexResult.rateLimitPct >= 95) return true;
  // 2. error message keywords
  const errorLower = (codexResult.error || '').toLowerCase();
  if (errorLower.includes('rate limit') || errorLower.includes('quota') || errorLower.includes('usage limit')) return true;
  // 3. HTTP 429 in JSONL
  if ((codexResult.output || '').includes('"status":429')) return true;
  // 4. Timeout with no output
  if (errorLower.includes('timed out') && !(codexResult.output || '').trim()) return true;
  return false;
}
```

### Verified Pattern: runMinimax() Return Object
Source: `~/.claude/hooks/minimax-exec.js` (verified on disk)
```javascript
// Successful result:
{
  success: true,
  text: '<think>\n...reasoning...\n</think>\n\nactual file content here',
  tokens: { input_tokens: 500, cached_input_tokens: 0, output_tokens: 800 },
  cost: 0.001110,  // USD
  error: null
}

// Key: strip <think>...</think> before writing to disk
const thinkEnd = result.text.indexOf('</think>');
const fileContent = thinkEnd !== -1
  ? result.text.slice(thinkEnd + '</think>'.length).trim()
  : result.text;
```

### Verified Pattern: [CODEX_RESULT] Marker for Token Logger
Source: `~/.claude/hooks/codex-token-logger.js` (verified on disk)
```
[CODEX_RESULT] {"model":"minimax-m2.7","source":"api-fallback","task_type":"execution","tokens":{"input_tokens":500,"cached_input_tokens":0,"output_tokens":800,"reasoning_output_tokens":0},"rate_limit_pct":null}
```
The logger reads `data.tool_result` from PostToolUse stdin, looks for `[CODEX_RESULT]`, and parses the JSON after it. Emit this as the last line of the Bash tool's output.

### Verified Pattern: Codex CLI --full-auto --json Invocation
Source: `codex exec --help` output (verified on installed v0.118.0)
```bash
# --full-auto is a convenience alias for: -a on-request --sandbox workspace-write
# --json prints events to stdout as JSONL
codex exec --full-auto --json --model gpt-5.4 "Create file src/hello.js with..."
```
`--full-auto` enables workspace-write sandbox (files can be created/modified) and sets approval mode to on-request. This is what `runCodexExec()` invokes internally.

### Verified Pattern: D-05 Model Config (Already Satisfied)
Source: `gsd-tools.cjs init execute-phase 13` output (verified live)
```json
{ "executor_model": "sonnet" }
```
The `balanced` profile maps `gsd-executor` to `sonnet` in `model-profiles.md`. No `.planning/config.json` change needed for this project.

---

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| gsd-executor writes code directly (Opus tokens) | gsd-executor generates handoff spec, delegates to Codex | Phase 13 | ~$10-15/day → ~$1-3/day for code generation |
| Single model execution | Three-tier fallback: Codex CLI → MiniMax API → user prompt | Phase 8 + Phase 13 | Zero code-generation Opus spend when Codex is available |
| Opus used for all subagents | Sonnet for executor (already configured in `balanced` profile) | model-profiles.md | 10x cost reduction for executor model calls |

**Deprecated/outdated:**
- Opus executor pattern: The old assumption that "executors follow instructions so Sonnet is fine" now also applies to the code-writing step — Codex CLI replaces Opus for all code writing.
- `runWithFallback()` for execution tasks: Works for review tasks; not suitable for execution tasks due to the MiniMax file-content-injection requirement.

---

## Open Questions

1. **Multi-file task handling**
   - What we know: D-06 says per-task, not per-file. Tasks in PLAN.md can touch multiple files.
   - What's unclear: Should the executor split a multi-file task into one Codex call per file, or send all files in one call?
   - Recommendation: One Codex call per file (Claude's discretion). Simpler validation, clearer error attribution. Batch only when files are inseparable.

2. **MiniMax API key availability in Claude Code subprocess env**
   - What we know: `MINIMAX_API_KEY` is not set in the current shell env. `runMinimax()` handles absence gracefully (returns error, triggers D-11 user prompt).
   - What's unclear: Whether the user will set the key before Phase 13 executes. The MiniMax fallback is only needed when Codex is rate-limited.
   - Recommendation: Include a plan task that verifies the key is accessible from a hook subprocess context. This can be the Phase 8 connectivity test script.

3. **Codex output validation depth**
   - What we know: D-02 specifies a verification command per handoff spec. Claude's discretion determines depth.
   - What's unclear: When should the executor run tests vs. just grep verification?
   - Recommendation: For `create` tasks, grep for the primary function/export name. For `modify` tasks, grep for the modified pattern. Tests only if the handoff spec explicitly says the task includes test changes. Keep validation fast (< 5s per task).

4. **runCodexExec() spec size limit**
   - What we know: `codex exec` accepts the prompt as a CLI argument (string). Very long specs could hit OS arg length limits (ARG_MAX ~2MB on Linux).
   - What's unclear: At what spec length does this become a practical concern for Phase 13 tasks?
   - Recommendation: For handoff specs over ~4KB, write to a temp file and pipe to stdin: `codex exec --full-auto --json - < /tmp/spec.txt`. The `codex exec` help confirms stdin is supported when `-` is used as the prompt.

---

## Environment Availability

| Dependency | Required By | Available | Version | Fallback |
|------------|------------|-----------|---------|----------|
| `codex` CLI | D-06 primary | Yes | 0.118.0 | MiniMax API (D-08) |
| `MINIMAX_API_KEY` | D-08/D-10 fallback | No (not set) | N/A | D-11 user prompt |
| Node.js | Hook scripts | Yes | v22.22.0 | N/A |
| `minimax-exec.js` | `runWithFallback`, `isCodexRateLimited` | Yes (Phase 8) | 1.1.0 | N/A |
| `codex-exec.js` | `runCodexExec`, `parseCodexTokens` | Yes (Phase 1) | 1.1.0 | N/A |

**Missing dependencies with no fallback:** None that block execution. `MINIMAX_API_KEY` absence is handled by D-11 (user prompt), which is an intentional design choice.

**Missing dependencies with fallback:** `MINIMAX_API_KEY` — if absent when MiniMax fallback is triggered, `runMinimax()` returns `{ success: false }`, which routes to D-11 user prompt. This is correct behavior, not a failure.

---

## Project Constraints (from CLAUDE.md)

Extracted actionable directives from `./CLAUDE.md`:

- **Security:** Never expose API keys in plaintext; use environment variables; bind services to 127.0.0.1 — affects how MINIMAX_API_KEY is referenced (env var only, never logged)
- **Compatibility:** Must work with existing GSD and Superpowers plugin versions without breaking current workflows — `gsd-executor.md` modifications must preserve all existing protocol sections (D-07)
- **Runtime:** Codex CLI runs locally in terminal; API calls use OpenAI SDK — invocation pattern is `codex exec --full-auto --json`, not API
- **Orchestration:** Opus always remains the primary orchestrator; Codex never makes architectural decisions autonomously — the gsd-executor becomes Sonnet, Codex handles implementation
- **Budget:** $20/mo ChatGPT Plus; $15/day max API spend; prefer CLI over API billing — Codex CLI is free via subscription; MiniMax is fallback only
- **MiniMax data privacy:** Don't send credentials or PII — handoff specs must not include API keys or personal data in file contents
- **Hook language:** Node.js (existing pattern) — any new helper scripts for this phase must be Node.js
- **What NOT to use:** `OpenAI Agents SDK`, `Claude Flow`, `LangChain` — no framework dependencies; use existing `runCodexExec()` and `runMinimax()` directly
- **`async: true` hooks for review:** Review needs to inject context synchronously — not applicable here (this is agent-internal execution, not a hook event), but confirms the pattern of synchronous validation after Codex completes
- **GSD Workflow Enforcement:** Use GSD entry points; do not make direct repo edits outside a GSD workflow — the phase plan must use `/gsd:execute-phase` and not direct file edits
- **Node.js version:** v22.22.0 installed — `fs.glob()` returns AsyncIterator, use `for await...of`. Not relevant to Phase 13 (no glob usage), but noted.
- **`--no-verify` in parallel mode:** Parallel executor agents use `--no-verify` on git commits. Phase 13 uses sequential task execution, so this is N/A per task, but relevant if multiple plans run in parallel within this phase.

---

## Sources

### Primary (HIGH confidence)
- `~/.claude/hooks/codex-exec.js` — `runCodexExec()` signature, result object shape, timeout behavior
- `~/.claude/hooks/minimax-exec.js` — `runWithFallback()`, `runMinimax()`, `isCodexRateLimited()` implementations
- `~/.claude/hooks/codex-pricing.js` — pricing constants, `computeCodexCostStrict()` for minimax-m2.7
- `~/.claude/hooks/codex-token-logger.js` — `[CODEX_RESULT]` marker format and parsing logic
- `~/.claude/agents/gsd-executor.md` — complete agent definition, execution protocol, what changes vs what stays
- `~/.claude/get-shit-done/references/model-profiles.md` — `balanced` profile maps gsd-executor to sonnet
- `codex exec --help` output — confirmed `--full-auto`, `--json`, `--model`, stdin support
- `gsd-tools.cjs init execute-phase 13` — confirmed `executor_model: "sonnet"` (D-05 already satisfied)

### Secondary (MEDIUM confidence)
- `minimax-m2.7-synthesis.md` §6 — architecture diagram and routing table confirmed against code
- `.planning/STATE.md` — Phase 8 decisions confirm `runWithFallback()` design intent
- `./CLAUDE.md` (project) — constraints cross-referenced against planned approach

### Tertiary (LOW confidence)
- None — all critical claims verified against code on disk or live CLI output

---

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH — all files verified on disk, versions confirmed live
- Architecture: HIGH — pattern derived from existing code, not speculation
- Pitfalls: HIGH — derived from reading actual implementation code and Phase 8 decisions
- Environment availability: HIGH — confirmed via live tool invocation

**Research date:** 2026-04-03
**Valid until:** 2026-05-03 (Codex CLI version may change; verify `codex --version` before executing)
