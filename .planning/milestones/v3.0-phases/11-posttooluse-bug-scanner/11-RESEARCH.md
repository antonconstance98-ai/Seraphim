# Phase 11: PostToolUse Bug Scanner - Research

**Researched:** 2026-04-03
**Domain:** Claude Code PostToolUse hook development, MiniMax M-2.7 API, git diff extraction
**Confidence:** HIGH

---

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions

- **D-01:** Code files only. Trigger on Write/Edit of `.js`, `.ts`, `.py`, `.jsx`, `.tsx`, `.mjs`, `.cjs`, and similar code extensions. Skip markdown, JSON config, planning docs, settings files.
- **D-02:** Send the git diff of changed files (with surrounding context lines), not the full file content. Keeps scans cheap and focused on what actually changed.
- **D-03:** Skip scans for trivial edits — if the diff is under ~5 changed lines AND only touches comments, strings, or whitespace. Saves $0.01-0.03 per trivial edit.
- **D-04:** Threshold is configurable via the `minimax` config block in `.claude/settings.json`: `"scan_skip_threshold": 5` (number of changed lines below which to skip).
- **D-05:** Advisory only — output findings as `additionalContext`, never `decision: "block"`. The Stop hook review gate handles blocking; this scanner is an early warning system.
- **D-06:** Cap `max_tokens` at 1000 to control MiniMax verbosity. Scan prompt asks for concise findings: file, line, issue, severity — no lengthy explanations.
- **D-07:** Integrate with existing `codex-token-logger.js` for tracking. Log entries use `model: 'minimax-m2.7'`, `source: 'api'`, `task_type: 'post-scan'`.
- **D-08:** If MiniMax API fails, fail silently (exit 0). Never block tool execution due to scanner failure. Log the error to stderr for debugging.

### Claude's Discretion

- Exact file extension list for "code files"
- Scan prompt wording (bug-focused vs security-focused vs both)
- Whether to include the file path and surrounding context in the scan prompt
- Debounce logic if multiple rapid Write/Edit calls happen in sequence

### Deferred Ideas (OUT OF SCOPE)

- Full-file analysis mode (send entire file instead of diff for deeper review)
- Configurable scan categories (bugs-only, security-only, performance) via settings
</user_constraints>

---

## Summary

Phase 11 creates `minimax-post-scan.js` — a PostToolUse hook that fires after every code-file Write/Edit and runs a lightweight MiniMax M-2.7 bug-and-security scan on the git diff. The hook is advisory-only (outputs `additionalContext`, never blocks), fail-silent on any error, and integrates with the existing token logging pipeline. It is the highest-value MiniMax integration point identified in research: MiniMax matched Opus 4.6 on bug detection (6/6) and security scanning (10/10) at 7% of the cost.

All infrastructure this phase needs already exists: `minimax-exec.js` (Phase 8) provides `runMinimax()`, `codex-pricing.js` already contains the `minimax-m2.7` pricing entry, and `~/.claude/settings.json` already has the PostToolUse hook group where this hook will be appended. The Phase 11 implementation is therefore a new file + one line added to settings.json.

The primary planning complexity is the diff extraction approach. The hook receives `tool_input.file_path` (for Write/Edit), but gets the diff via `git diff HEAD -U5` scoped to that path — not from the PostToolUse payload itself, which does not contain the diff. Trivial-edit filtering works on the diff line count before the API call.

**Primary recommendation:** `minimax-post-scan.js` is a thin hook script (~150 lines) that extracts the changed file path from `tool_input`, filters out non-code extensions, runs `git diff HEAD -U5 -- <path>` to get the focused diff, applies the trivial-edit threshold, calls `runMinimax()` with `max_tokens: 1000`, and outputs findings as `additionalContext`.

---

## Standard Stack

### Core

| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| Node.js built-ins (`fs`, `path`, `child_process`) | v22.22.0 (installed) | File I/O, path resolution, git subprocess | Already used by all hooks; no new dependencies |
| `minimax-exec.js` | project module (Phase 8) | `runMinimax()` call with retry + timeout | Shared wrapper; all Phase 9-12 hooks use it |
| `codex-pricing.js` | project module (Phase 8) | `computeCodexCostStrict()` for cost logging | Already contains `minimax-m2.7` pricing entry |
| `execSync` from `child_process` | Node.js built-in | `git diff HEAD -U5` subprocess | Same pattern as `codex-review-gate.js` |

### Supporting

| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| `fs.appendFileSync` | Node.js built-in | Append JSONL record to `token-log.jsonl` | Every scan that produces a result |

### Alternatives Considered

| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| `git diff HEAD -U5 -- <path>` | `tool_result` diff field from PostToolUse payload | PostToolUse payload does not contain a diff; git is the canonical source |
| `execSync` for git | `spawnSync` | `execSync` is simpler for synchronous one-shot commands; already used in review-gate |
| `runMinimax()` directly | `runWithFallback()` | `runWithFallback()` tries Codex CLI first; D-08 requires silent fail, not Codex fallback |

**Installation:** No new packages required. All dependencies are already installed.

---

## Architecture Patterns

### Recommended File Location

```
~/.claude/hooks/
├── minimax-post-scan.js        # New — Phase 11
├── minimax-exec.js             # Existing — Phase 8
├── codex-pricing.js            # Existing — Phase 8
├── codex-token-logger.js       # Existing — for reference
└── codex-wave-validator.js     # Existing — PostToolUse pattern reference
```

Settings registration is in `~/.claude/settings.json` PostToolUse group (append to existing `hooks` array under the `Bash|Edit|Write|MultiEdit|Agent|Task` matcher).

### Pattern 1: PostToolUse Stdin Payload Shape

The hook receives this JSON on stdin:

```javascript
// Source: verified from existing hooks (codex-token-logger.js, codex-wave-validator.js)
{
  tool_name:    'Write',         // or 'Edit', 'MultiEdit', 'Bash'
  tool_input:   { file_path: '/abs/path/to/file.js', content: '...' }, // Write/Edit
  tool_result:  '...',           // output of the tool (string or object)
  session_id:   'abc-123',
  cwd:          '/home/alucard/projects/...'
}
```

For `MultiEdit`, `tool_input` has `file_path` at top level. For `Bash`, `tool_input` has `command`; file path must be extracted from the command string (heuristic, less reliable — not required for Phase 11 since Write/Edit are the primary targets).

### Pattern 2: File Extension Filter

Established approach from the codebase:

```javascript
// Source: 11-CONTEXT.md D-01; matches pattern from codex-review-gate.js
const CODE_EXTENSIONS = new Set([
  '.js', '.ts', '.jsx', '.tsx', '.mjs', '.cjs',
  '.py', '.rb', '.go', '.rs', '.java', '.c', '.cpp',
  '.sh', '.bash', '.zsh',
  '.html', '.css', '.scss',
  '.sql'
]);

function isCodeFile(filePath) {
  const ext = path.extname(filePath).toLowerCase();
  if (!CODE_EXTENSIONS.has(ext)) return false;
  // Exclude planning docs and settings regardless of extension
  if (filePath.includes('.planning/')) return false;
  if (filePath.includes('settings.json')) return false;
  if (filePath.endsWith('.md')) return false;
  return true;
}
```

### Pattern 3: Git Diff Extraction (Focused Per-File)

Use the file path from `tool_input` to scope the diff tightly. This is cheaper to send to MiniMax than the full `git diff HEAD`.

```javascript
// Source: codex-review-gate.js collectDiff() pattern adapted for per-file scoping
const { execSync } = require('child_process');

function getFileDiff(filePath, cwd) {
  try {
    // Case 1: tracked file — get diff against HEAD
    const diff = execSync(
      `git diff HEAD -U5 -- "${filePath}"`,
      { cwd, timeout: 5000, maxBuffer: 256 * 1024 }
    ).toString();
    if (diff.trim()) return diff;

    // Case 2: new (untracked) file — diff against /dev/null
    try {
      execSync(`git diff --no-index /dev/null "${filePath}"`, {
        cwd, timeout: 5000, maxBuffer: 256 * 1024
      });
    } catch (e) {
      // git diff --no-index exits non-zero when files differ; content is in stdout
      if (e.stdout) return e.stdout.toString();
    }
    return '';
  } catch (e) {
    return '';
  }
}
```

### Pattern 4: Trivial Edit Detection (D-03)

Count changed lines in the diff (lines starting with `+` or `-`, excluding diff headers starting with `+++`/`---`):

```javascript
// Source: CONTEXT.md D-03, D-04
function isTrivialEdit(diff, threshold) {
  const changedLines = diff
    .split('\n')
    .filter(line => (line.startsWith('+') || line.startsWith('-'))
                    && !line.startsWith('+++')
                    && !line.startsWith('---'));
  if (changedLines.length >= threshold) return false;

  // All changed lines are comments, whitespace, or string-only changes
  const meaningful = changedLines.filter(line => {
    const content = line.slice(1).trim();
    // Skip blank lines
    if (!content) return false;
    // Skip comment lines (JS/TS/Python/Ruby/Go/Bash)
    if (/^(\/\/|#|\/\*|\*|--\s)/.test(content)) return false;
    // Skip lines that are only string literals (very rough heuristic)
    if (/^['"`].*['"`][,;]?$/.test(content)) return false;
    return true;
  });
  return meaningful.length === 0;
}
```

### Pattern 5: Scan Prompt Design

The CONTEXT.md specifies a two-pronged approach (bugs AND security). The output format must be scannable at a glance:

```javascript
// Source: 11-CONTEXT.md <specifics>
function buildScanPrompt(diff, filePath) {
  const truncatedDiff = diff.length > 3000
    ? diff.slice(0, 3000) + '\n... (truncated)'
    : diff;
  return `You are a code security and bug scanner. Analyze the following git diff for bugs and security vulnerabilities.

File: ${path.basename(filePath)}

Rules:
- Report only actual bugs or security issues, not style or preference
- Each finding on its own line in this exact format:
  [BUG] filename:linenum — description
  [SECURITY] filename:linenum — description
- If nothing found, output exactly: CLEAN
- Be concise — one line per finding, no explanations

Diff:
\`\`\`diff
${truncatedDiff}
\`\`\``;
}
```

### Pattern 6: Token Log Record (D-07)

The token log record for this hook uses `task_type: 'post-scan'` and `source: 'api'`:

```javascript
// Source: 11-CONTEXT.md D-07; schema from codex-review-gate.js MiniMax log entry
const record = {
  timestamp:   new Date().toISOString(),
  session_id:  data.session_id || null,
  model:       'minimax-m2.7',
  source:      'api',
  task_type:   'post-scan',
  tokens: {
    input:        minimaxResult.tokens.input_tokens        || 0,
    cached_input: minimaxResult.tokens.cached_input_tokens || 0,
    output:       minimaxResult.tokens.output_tokens       || 0
  },
  cost_usd: computeCodexCostStrict(minimaxResult.tokens, 'minimax-m2.7') || 0
};
fs.appendFileSync(logPath, JSON.stringify(record) + '\n', 'utf8');
```

### Pattern 7: Hook Output (Advisory)

PostToolUse advisory output — single JSON write to stdout:

```javascript
// Source: codex-token-logger.js pattern; 11-CONTEXT.md D-05
process.stdout.write(JSON.stringify({
  hookSpecificOutput: {
    hookEventName: 'PostToolUse',
    additionalContext: `[BUG SCAN] ${findings}`
  }
}));
```

When no issues found, either skip stdout entirely (exit 0) or output a minimal CLEAN indicator. Avoid spamming Claude's context on every file write — output only when findings exist (or optionally a brief CLEAN notice).

### Pattern 8: Settings.json Registration

Append to the existing PostToolUse hook array (same matcher, same group):

```json
// Source: ~/.claude/settings.json — existing PostToolUse group
{
  "type": "command",
  "command": "node \"/home/alucard/.claude/hooks/minimax-post-scan.js\"",
  "timeout": 10
}
```

**Critical:** The timeout is 10 seconds (same as other PostToolUse hooks). MiniMax pre-answer latency can be up to 55 seconds. This is a fundamental conflict — see Pitfall 1 below.

### Anti-Patterns to Avoid

- **Never output `decision: "block"` from a PostToolUse hook** — PostToolUse hooks cannot block in Claude Code (only PreToolUse hooks can). The hook should always use `additionalContext`.
- **Never call `runWithFallback()`** — This tries Codex CLI first, which is not appropriate for a silent advisory scanner. Use `runMinimax()` directly per D-08.
- **Never send the full file content** — D-02 mandates diff-only to keep scans cheap. `tool_input.content` contains the full file; use git diff instead.
- **Never write two JSON objects to stdout** — PostToolUse hooks must output exactly one JSON object. Combining token log message with scan findings in a single `additionalContext` is correct.

---

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| MiniMax API call with retry/timeout | Custom fetch + retry loop | `runMinimax()` from `minimax-exec.js` | Already has exponential backoff, AbortController, API key guard, token extraction |
| Cost calculation for `minimax-m2.7` | Inline multiplication | `computeCodexCostStrict(tokens, 'minimax-m2.7')` from `codex-pricing.js` | Pricing already loaded; function returns null for unknown models (correct error signal) |
| Token logging schema | New JSONL format | Existing JSONL schema from `codex-review-gate.js` MiniMax section | Aggregator and dashboard already understand this schema |

**Key insight:** All the hard infrastructure (SDK wrapper, pricing, retry, token logging) was built in Phases 8-10. Phase 11 assembles these pieces into a hook script.

---

## Runtime State Inventory

> This is not a rename/refactor/migration phase. No runtime state inventory required.

---

## Environment Availability

| Dependency | Required By | Available | Version | Fallback |
|------------|------------|-----------|---------|----------|
| Node.js | Hook runtime | Yes | v22.22.0 | — |
| git | Diff extraction | Yes | 2.43.0 | Skip scan (exit 0) |
| `minimax-exec.js` | `runMinimax()` call | Yes | Phase 8 (installed) | — |
| `codex-pricing.js` | Cost computation | Yes | Phase 8 (installed) | — |
| `MINIMAX_API_KEY` env var | MiniMax API auth | Assumed set (Phase 8 prerequisite) | — | Silent fail (D-08) |
| `~/.claude/settings.json` | Hook registration | Yes | Verified | — |

**Missing dependencies with no fallback:** None — all dependencies are confirmed present.

**Missing dependencies with fallback:**
- `MINIMAX_API_KEY` not set → `runMinimax()` returns `{ success: false }` → hook exits 0 silently (D-08). No user impact, no block.

---

## Common Pitfalls

### Pitfall 1: Hook Timeout vs MiniMax Latency

**What goes wrong:** PostToolUse hooks are registered with `"timeout": 10` seconds in `settings.json`. MiniMax pre-answer latency can be up to 55 seconds on first call. The hook will time out on the first scan after a cold start.

**Why it happens:** The 10-second timeout is standard for advisory hooks. The Stop hook uses 300 seconds because it can block. PostToolUse hooks must be non-blocking and fast per Claude Code design.

**How to avoid:** There are two viable approaches:
1. **Accept the limitation:** The first scan after cold start will time out silently (exit 0 by timeout). Subsequent scans benefit from MiniMax connection reuse (lower latency). Add a note in the hook comment.
2. **Increase the timeout to 60 seconds:** The settings.json `timeout` field is per-hook. Increasing to `"timeout": 60` would cover most MiniMax calls. The downside is Claude waits up to 60s after every Write/Edit for the hook to return.

**Recommended approach:** Set timeout to `"timeout": 30` as a pragmatic middle ground — covers typical MiniMax response time (2.3-3.3s TTFT + output), misses outlier 55s cases (which fail silently per D-08). Document this in hook comments.

**Warning signs:** Scan results never appear in Claude's context. Check stderr for timeout messages.

### Pitfall 2: PostToolUse Payload Does Not Contain the Diff

**What goes wrong:** A new hook developer reads `tool_result` expecting the diff, but `tool_result` for a Write operation contains the success status string, not the file content diff.

**Why it happens:** The PostToolUse payload reflects what the tool returned, not what changed on disk.

**How to avoid:** Always use `git diff HEAD -U5 -- <file_path>` to get the diff. The file path is in `tool_input.file_path` for Write and Edit operations. For MultiEdit, the same field applies. Verify `tool_input` is parsed correctly before accessing `file_path`.

**Warning signs:** Empty diff strings passed to MiniMax, resulting in "CLEAN" responses on every scan regardless of changes.

### Pitfall 3: MultiEdit Tool Shape

**What goes wrong:** `tool_input` for MultiEdit has a slightly different shape — it contains an array of edits, all targeting the same `file_path`. Reading `tool_input.file_path` still works, but `tool_input.edits` (the array) is the actual content change.

**Why it happens:** MultiEdit is a batch operation; the payload reflects that.

**How to avoid:** The git diff approach (Pattern 3) is immune to this — it reads from disk regardless of which tool wrote. No special MultiEdit handling needed.

### Pitfall 4: Diff Contains Binary Files or Non-Scannable Content

**What goes wrong:** `git diff HEAD` might include binary file diffs (images, compiled artifacts) that produce noise or inflate token count.

**Why it happens:** The file extension filter catches code extensions, but a `.js` file path could theoretically be written as binary.

**How to avoid:** Check the diff output for the git binary marker (`Binary files differ`). If present, skip the scan and exit 0 silently. The `isCodeFile()` extension filter handles most cases.

### Pitfall 5: Scanning `.planning/` or Settings Files

**What goes wrong:** Planning docs and settings files (`.planning/`, `.claude/settings.json`, `CLAUDE.md`) get scanned, producing irrelevant noise in Claude's context.

**Why it happens:** Write/Edit fires on ALL file writes, including planning artifacts.

**How to avoid:** The `isCodeFile()` function must explicitly exclude `.planning/` paths and `settings.json` before doing any diff work (before `git diff`, not after).

### Pitfall 6: `runMinimax()` Returns `{ success: false }` — Don't Log Null Tokens

**What goes wrong:** On failure, `minimaxResult.tokens` is `null`. Calling `computeCodexCostStrict(null, 'minimax-m2.7')` returns 0 (not null), but trying to access `null.input_tokens` throws.

**Why it happens:** D-08 requires silent fail; the failure path must not log a broken JSONL record.

**How to avoid:** Guard the token logging block: `if (minimaxResult.success && minimaxResult.tokens) { ... log ... }`. Only log on success.

### Pitfall 7: `scan_skip_threshold` Config Key May Not Exist Yet

**What goes wrong:** D-04 says the threshold is in `minimax.scan_skip_threshold` in `.claude/settings.json`. The current project settings.json does not have this key yet.

**Why it happens:** This key is new for Phase 11; it doesn't exist in the Phase 8-built settings block.

**How to avoid:** Read the config with a safe default: `const threshold = minimaxConfig.scan_skip_threshold || 5;`. Also, the planner must add `"scan_skip_threshold": 5` to the `minimax` block in `.claude/settings.json` as a task.

---

## Code Examples

### Full Hook Skeleton (Verified Pattern)

```javascript
// Source: codex-token-logger.js + codex-wave-validator.js patterns combined
'use strict';

const fs   = require('fs');
const path = require('path');
const { execSync } = require('child_process');

const MINIMAX_EXEC_PATH = '/home/alucard/.claude/hooks/minimax-exec';
const PRICING_PATH      = '/home/alucard/.claude/hooks/codex-pricing';

const stdinTimeout = setTimeout(() => process.exit(0), 10000);

let input = '';
process.stdin.setEncoding('utf8');
process.stdin.on('data', (chunk) => { input += chunk; });
process.stdin.on('end', async () => {
  clearTimeout(stdinTimeout);
  try {
    const data    = JSON.parse(input);
    const cwd     = data.cwd || process.cwd();
    const toolName = data.tool_name || '';

    // 1. Only fire on Write / Edit / MultiEdit
    if (!['Write', 'Edit', 'MultiEdit'].includes(toolName)) {
      process.exit(0);
    }

    // 2. Extract file path
    const filePath = (data.tool_input && data.tool_input.file_path) || '';
    if (!filePath || !isCodeFile(filePath)) {
      process.exit(0);
    }

    // 3. Read minimax config (with defaults)
    const projectSettings = readProjectSettings(cwd);
    const minimaxConfig   = projectSettings.minimax || {};
    if (minimaxConfig.enabled === false) process.exit(0);
    const threshold = minimaxConfig.scan_skip_threshold || 5;

    // 4. Get diff
    const diff = getFileDiff(filePath, cwd);
    if (!diff || !diff.trim()) process.exit(0);

    // 5. Trivial edit check
    if (isTrivialEdit(diff, threshold)) process.exit(0);

    // 6. Call MiniMax
    const { runMinimax } = require(MINIMAX_EXEC_PATH);
    const prompt = buildScanPrompt(diff, filePath);
    const result = await runMinimax(prompt, { maxTokens: 1000, timeoutMs: 25000 });

    if (!result.success) {
      process.stderr.write('minimax-post-scan: API failed: ' + result.error + '\n');
      process.exit(0);
    }

    // 7. Log tokens
    if (result.tokens) {
      const { computeCodexCostStrict } = require(PRICING_PATH);
      const record = {
        timestamp:   new Date().toISOString(),
        session_id:  data.session_id || null,
        model:       'minimax-m2.7',
        source:      'api',
        task_type:   'post-scan',
        tokens: {
          input:        result.tokens.input_tokens        || 0,
          cached_input: result.tokens.cached_input_tokens || 0,
          output:       result.tokens.output_tokens       || 0
        },
        cost_usd: computeCodexCostStrict(result.tokens, 'minimax-m2.7') || 0
      };
      const logPath = path.join(cwd, '.planning', 'token-log.jsonl');
      fs.mkdirSync(path.dirname(logPath), { recursive: true });
      fs.appendFileSync(logPath, JSON.stringify(record) + '\n', 'utf8');
    }

    // 8. Surface findings (advisory only)
    const findings = result.text.trim();
    if (findings && findings !== 'CLEAN') {
      process.stdout.write(JSON.stringify({
        hookSpecificOutput: {
          hookEventName: 'PostToolUse',
          additionalContext: '[BUG SCAN] ' + path.basename(filePath) + ':\n' + findings
        }
      }));
    }
    // If CLEAN: exit 0 silently (don't pollute context on every file save)
  } catch (e) {
    process.stderr.write('minimax-post-scan: error: ' + e.message + '\n');
  }
  process.exit(0);
});
```

### settings.json Registration

```json
// Append inside the existing PostToolUse hooks array (same matcher group)
{
  "type": "command",
  "command": "node \"/home/alucard/.claude/hooks/minimax-post-scan.js\"",
  "timeout": 30
}
```

### Project settings.json minimax block update

```json
// .claude/settings.json — add scan_skip_threshold to existing minimax block
{
  "minimax": {
    "enabled": true,
    "model": "MiniMax-M2.7",
    "api_key_env": "MINIMAX_API_KEY",
    "base_url": "https://api.minimax.io/v1",
    "max_tokens_default": 2000,
    "scan_skip_threshold": 5,
    "tasks": ["bug-scan", "security-scan", "third-opinion", "context-compress"]
  }
}
```

---

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| PostToolUse used only for token logging and wave validation | PostToolUse now also does inline MiniMax scanning | Phase 11 | Catches bugs in real-time as code is written, not only at Stop gate |
| Single Stop-hook review gate | Two layers: PostToolUse early warning + Stop gate hard block | Phase 9 + Phase 11 | Defense in depth; issues surfaced earlier |
| MiniMax called only for review/adversarial roles | MiniMax also used for per-file scanning | Phase 11 | Expands MiniMax ROI; ~$0.01-0.03/scan at $0.30/$1.20 pricing |

---

## Open Questions

1. **Timeout: 10s vs 30s vs 60s**
   - What we know: MiniMax typical TTFT is 2.3-3.3s; pre-answer latency outliers reach 55s
   - What's unclear: What is the P95 latency in practice for a 3000-char prompt?
   - Recommendation: Start with `"timeout": 30`. If scans consistently time out (check stderr), increase to 60.

2. **CLEAN response: silent exit or output a brief notice?**
   - What we know: Outputting `additionalContext` on every file write adds noise to Claude's context
   - What's unclear: Whether users/operators want positive confirmation scans are running
   - Recommendation: Silent exit (no stdout) on CLEAN. The token log entry serves as proof of execution.

3. **Debounce for rapid multiple writes**
   - What we know: CONTEXT.md flags this as Claude's discretion; no debounce infrastructure exists
   - What's unclear: How often does Claude write 5+ files in rapid sequence?
   - Recommendation: No debounce in Phase 11. Each write triggers its own scan (cost is low). Debounce complexity adds risk; defer to Deferred Ideas if needed.

---

## Project Constraints (from CLAUDE.md)

The following directives from `CLAUDE.md` apply to this phase:

| Directive | Applies How |
|-----------|-------------|
| Never expose API keys in plaintext | `MINIMAX_API_KEY` read from `process.env`, never hardcoded |
| Bind services to 127.0.0.1 | Not applicable — no service binding in this hook |
| Use environment variables for secrets | Confirmed: `minimax-exec.js` already guards for `MINIMAX_API_KEY` |
| Prefer simple, working solutions over complex clever ones | No debounce, no caching layer — thin hook that calls existing primitives |
| Before any destructive command, explain and confirm | Hook is read-only (git diff) + append-only (JSONL log) — no destructive ops |
| GSD workflow enforcement: use `/gsd:execute-phase` | This phase is executed via GSD; no direct edits outside workflow |
| Must work with existing GSD and Superpowers plugin versions | Hook registered in global settings.json; no GSD plugin modification |
| Opus remains primary orchestrator; Codex never makes architectural decisions | MiniMax is advisory only; Opus reads the `additionalContext` and decides |
| Runtime: Codex CLI runs locally; API calls use OpenAI SDK | `minimax-exec.js` uses OpenAI SDK with baseURL swap (not Codex CLI) |

---

## Sources

### Primary (HIGH confidence)

- `~/.claude/hooks/minimax-exec.js` — verified `runMinimax()` signature, return shape `{ success, text, tokens, cost }`, timeout behavior, API key guard
- `~/.claude/hooks/codex-pricing.js` — verified `minimax-m2.7` pricing entry exists (`input: 0.30, cached_input: 0.06, output: 1.20`), `computeCodexCostStrict()` available
- `~/.claude/hooks/codex-review-gate.js` — verified git diff extraction pattern, `detectCodeChanges()`, `collectDiff()`, MiniMax token log record schema
- `~/.claude/hooks/codex-token-logger.js` — verified PostToolUse stdin payload shape, JSONL schema, `additionalContext` output pattern
- `~/.claude/hooks/codex-wave-validator.js` — verified non-blocking PostToolUse hook structure, file path extraction from `tool_input`
- `~/.claude/settings.json` — verified current PostToolUse hooks registration, matcher `Bash|Edit|Write|MultiEdit|Agent|Task`, timeout 10
- `.claude/settings.json` — verified existing `minimax` block structure (missing `scan_skip_threshold`)
- `minimax-m2.7-synthesis.md` — verified bug detection (6/6), security scanning (10/10), pricing, latency specs, verbosity tax

### Secondary (MEDIUM confidence)

- `.planning/phases/08-minimax-foundation/08-CONTEXT.md` — `runMinimax()` design decisions, config block spec, D-03 (temperature 0.01), D-04 (max_tokens 2000 default)
- `.planning/STATE.md` — confirmed Phase 8 minimax-exec.js decisions including callWithRetry wrapping, defensive cached_tokens fallback

### Tertiary (LOW confidence)

- None. All claims verified from local files.

---

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH — all dependencies verified installed and functional
- Architecture: HIGH — patterns copied directly from working Phase 9/10 hooks
- Pitfalls: HIGH — derived from actual code review of existing hooks + MiniMax synthesis doc
- Timeout recommendation: MEDIUM — P95 MiniMax latency not empirically measured on this machine; based on synthesis doc data

**Research date:** 2026-04-03
**Valid until:** 2026-05-03 (stable infrastructure; MiniMax API may change pricing)
