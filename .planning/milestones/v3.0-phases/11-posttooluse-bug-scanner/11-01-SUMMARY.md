---
phase: 11-posttooluse-bug-scanner
plan: "01"
subsystem: hooks
tags: [minimax, post-tool-use, bug-scan, security, advisory]
dependency_graph:
  requires:
    - ~/.claude/hooks/minimax-exec.js      # runMinimax() API wrapper
    - ~/.claude/hooks/codex-pricing.js     # computeCodexCostStrict() for cost logging
  provides:
    - ~/.claude/hooks/minimax-post-scan.js # PostToolUse bug/security scanner hook
  affects:
    - ~/.claude/settings.json              # PostToolUse hooks array (new 4th entry)
    - .claude/settings.json                # minimax.scan_skip_threshold config key added
tech_stack:
  added: []
  patterns:
    - PostToolUse hook stdin pattern (stdinTimeout + JSON.parse)
    - execFileSync array args for shell-injection-safe git subprocess calls
    - Lazy-require of minimax-exec and codex-pricing (only when scan proceeds)
    - advisory additionalContext output (never decision:block from PostToolUse)
key_files:
  created:
    - ~/.claude/hooks/minimax-post-scan.js   # 362-line PostToolUse hook
  modified:
    - ~/.claude/settings.json               # +4 lines: minimax-post-scan hook entry, timeout:30
    - .claude/settings.json                 # +1 line: scan_skip_threshold: 5 in minimax block
decisions:
  - "execFileSync array args (not execSync string) for git calls -- prevents shell injection via file paths with metacharacters"
  - "isTrivialEdit classifies only blank/whitespace/comment lines as trivial -- string literals are NOT trivial (URLs, SQL, regex are security-relevant)"
  - "Lazy-require minimax-exec and codex-pricing only after code-file and diff checks -- avoids SDK load on every non-code write"
  - "Strip <think> reasoning block before outputting additionalContext -- keeps advisory focused on actionable findings"
metrics:
  duration_seconds: 197
  completed_date: "2026-04-03"
  tasks_completed: 2
  tasks_total: 2
  files_created: 1
  files_modified: 2
---

# Phase 11 Plan 01: PostToolUse Bug Scanner Summary

**One-liner:** MiniMax M-2.7 PostToolUse bug/security scanner with execFileSync-safe git diff, trivial-edit skip, and advisory additionalContext output.

## What Was Built

`minimax-post-scan.js` is a PostToolUse hook that scans code file diffs with MiniMax M-2.7 after every Write, Edit, or MultiEdit operation. It is advisory-only, fail-silent, and runs as the 4th hook in the existing PostToolUse group (alongside gsd-context-monitor, codex-token-logger, and codex-wave-validator).

## Key Implementation Details

### Hook Architecture

The hook follows the standard PostToolUse stdin-reading pattern with a 10-second timeout guard. All logic is wrapped in a try/catch that exits 0 on any unexpected error. The outer catch ensures no error can block tool execution.

### 8 Locked Decisions (D-01 through D-08)

| Decision | Implementation |
|----------|----------------|
| D-01: Code-file filter | `isCodeFile()` with Set of 20 extensions, `toLowerCase()` for case normalization; excludes `/.planning/`, `/.claude/settings.json`, `.md` |
| D-02: Diff-based scanning | `getFileDiff()` with `execFileSync` array args; handles fresh repos via `rev-parse HEAD` → staged diff → untracked fallback |
| D-03: Trivial edit skip | `isTrivialEdit()` classifies only blank/whitespace/comment lines as trivial; string literals are NOT trivial |
| D-04: Configurable threshold | `scan_skip_threshold` read from project `.claude/settings.json`; default 5; `scan_skip_threshold: 5` added to project settings |
| D-05: Advisory output | `additionalContext` in `hookSpecificOutput`; silent on CLEAN, skip, error; no `decision:block` (not supported by PostToolUse) |
| D-06: Token cap | `maxTokens: 1000`, `timeoutMs: 25000` (leaves 5s margin within 30s hook timeout) |
| D-07: Token logging | `task_type: 'post-scan'`; logged only on `result.success && result.tokens`; uses `computeCodexCostStrict` |
| D-08: Fail-silent | `process.exit(0)` on API failure, config read error, diff failure, and outer catch |

### Security: execFileSync vs execSync

All git subprocess calls use `execFileSync(cmd, argsArray, opts)` instead of `execSync(cmdString)`. This prevents shell injection when file paths contain metacharacters like spaces, backticks, semicolons, or dollar signs.

### Fresh Repo Handling

The hook checks for `rev-parse HEAD` success before attempting `git diff HEAD`. On failure (no commits yet), it falls back to `git diff --cached` (staged files) then `git diff --no-index /dev/null` (untracked file). This prevents errors on brand-new repositories.

### Trivial Edit Detection

Changed lines are extracted from the diff (lines starting with `+` or `-`, excluding `+++`/`---` headers). If the count meets or exceeds the threshold (default 5), the file is always scanned. Below the threshold, the edit is trivial only if ALL changed lines are blank, whitespace-only, or match the comment pattern `/(\/\/|#|\/\*|\*|--\s)/`. String literals, function calls, expressions — all non-trivial.

### Advisory Output Format

When findings exist and the response is not `CLEAN`, the hook strips any `<think>` reasoning block and outputs:

```
[BUG SCAN] filename.js:
[BUG] filename.js:42 -- description
[SECURITY] filename.js:15 -- description
```

When CLEAN, skipped (non-code, trivial, no diff), or on API error: no stdout (silent).

## Settings Changes

**`~/.claude/settings.json`** — new 4th hook added to the `Bash|Edit|Write|MultiEdit|Agent|Task` PostToolUse group:
```json
{
  "type": "command",
  "command": "node \"/home/alucard/.claude/hooks/minimax-post-scan.js\"",
  "timeout": 30
}
```

**`.claude/settings.json`** — `scan_skip_threshold: 5` added to `minimax` block.

## Verification Results

All 9 end-to-end checks passed:
1. Syntax: `node -c` exits 0
2. Global settings: valid JSON
3. Project settings: valid JSON
4. Hook registered: 1 entry, no duplicates
5. Config present: `scan_skip_threshold: 5`
6. No shell injection: `execFileSync` confirmed
7. Fresh repo guard: `rev-parse HEAD` confirmed
8. `.md` skip test: exit 0, empty stdout
9. All 8 D-xx decisions: grep-verified

## Deviations from Plan

### Auto-fixed Issues

None — plan executed exactly as written.

One clarification applied (not a deviation): the `<think>` block stripping before `additionalContext` output. The plan specified outputting `result.text.trim()` as findings, but MiniMax responses include `<think>` reasoning blocks (per minimax-exec.js behavior documented in Phase 8). Stripping these before outputting keeps the advisory context focused on actionable `[BUG]`/`[SECURITY]` lines only. This is consistent with D-05 (advisory, concise) and the `CLEAN` response handling.

## Known Stubs

None. The hook is fully wired:
- `runMinimax` from `minimax-exec.js` (Phase 8, verified working)
- `computeCodexCostStrict` from `codex-pricing.js` (Phase 5, verified)
- Token log to `.planning/token-log.jsonl` in project `cwd`

## Commits

| Task | Commit | Description |
|------|--------|-------------|
| Task 1: minimax-post-scan.js | 9cb3758 | Create hook (outside repo -- empty commit) |
| Task 2: settings registration | 888bcb0 | Register hook, add scan_skip_threshold |

## Self-Check: PASSED

- `~/.claude/hooks/minimax-post-scan.js` exists (362 lines)
- Commit 9cb3758 exists (documentation commit, file outside repo)
- Commit 888bcb0 exists (`.claude/settings.json` modified)
- All 8 D-xx decisions verified with grep
- Both JSON files parse without errors
