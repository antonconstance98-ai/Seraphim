# Phase 11: PostToolUse Bug Scanner - Context

**Gathered:** 2026-04-03
**Status:** Ready for planning

<domain>
## Phase Boundary

Create a new PostToolUse hook (`minimax-post-scan.js`) that runs a lightweight MiniMax bug/security scan after every code file Write/Edit. Advisory only — never blocks. Catches issues in real-time as code is written, before the Stop hook review gate.

</domain>

<decisions>
## Implementation Decisions

### Scan scope
- **D-01:** Code files only. Trigger on Write/Edit of `.js`, `.ts`, `.py`, `.jsx`, `.tsx`, `.mjs`, `.cjs`, and similar code extensions. Skip markdown, JSON config, planning docs, settings files.
- **D-02:** Send the git diff of changed files (with surrounding context lines), not the full file content. Keeps scans cheap and focused on what actually changed.

### Trivial edit threshold
- **D-03:** Skip scans for trivial edits — if the diff is under ~5 changed lines AND only touches comments, strings, or whitespace. Saves $0.01-0.03 per trivial edit.
- **D-04:** Threshold is configurable via the `minimax` config block in `.claude/settings.json`: `"scan_skip_threshold": 5` (number of changed lines below which to skip).

### Output behavior
- **D-05:** Advisory only — output findings as `additionalContext`, never `decision: "block"`. The Stop hook review gate handles blocking; this scanner is an early warning system.
- **D-06:** Cap `max_tokens` at 1000 to control MiniMax verbosity. Scan prompt asks for concise findings: file, line, issue, severity — no lengthy explanations.

### Token logging
- **D-07:** Integrate with existing `codex-token-logger.js` for tracking. Log entries use `model: 'minimax-m2.7'`, `source: 'api'`, `task_type: 'post-scan'`.
- **D-08:** If MiniMax API fails, fail silently (exit 0). Never block tool execution due to scanner failure. Log the error to stderr for debugging.

### Claude's Discretion
- Exact file extension list for "code files"
- Scan prompt wording (bug-focused vs security-focused vs both)
- Whether to include the file path and surrounding context in the scan prompt
- Debounce logic if multiple rapid Write/Edit calls happen in sequence

</decisions>

<specifics>
## Specific Ideas

- The scan prompt should be two-pronged: "Find bugs AND security vulnerabilities in this diff" — MiniMax matched Opus on both (6/6 bugs, 10/10 vulns) in head-to-head testing
- Output format should be scannable by Opus at a glance: "[BUG] file.js:42 — off-by-one in loop bound" or "[SECURITY] auth.js:15 — SQL injection via unsanitized input"
- This is the highest-value MiniMax integration point per the research — cheap, accurate, catches issues before they accumulate

</specifics>

<canonical_refs>
## Canonical References

### Phase 8 Foundation (dependency)
- `.planning/phases/08-minimax-foundation/08-CONTEXT.md` — `runMinimax()` interface, pricing, config block

### Existing PostToolUse Hooks
- `~/.claude/hooks/codex-token-logger.js` — Token logging pattern to integrate with
- `~/.claude/hooks/codex-wave-validator.js` — Example of a non-blocking PostToolUse hook with background processing

### Hook Registration
- `~/.claude/settings.json` — PostToolUse hooks section, `Bash|Edit|Write|MultiEdit|Agent|Task` matcher

### Research
- `minimax-m2.7-synthesis.md` §2 "Head-to-Head Testing" — Bug detection 6/6, security vulns 10/10
- `minimax-m2.7-synthesis.md` §6 "Multi-Pass Review" — Cost analysis for per-scan economics

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `codex-wave-validator.js` — Non-blocking PostToolUse hook pattern: detects file writes, runs analysis, stores results, surfaces on next tool use.
- `codex-token-logger.js` — Token logging JSONL schema and append pattern.

### Established Patterns
- PostToolUse hooks receive `{ tool_name, tool_input, tool_result, session_id, cwd }` on stdin
- 10-second timeout for PostToolUse hooks (registered in settings.json)
- Fail-open: `process.exit(0)` on any error
- Advisory output via `additionalContext` in stdout JSON

### Integration Points
- New hook registered in `~/.claude/settings.json` under the existing PostToolUse group (same matcher: `Bash|Edit|Write|MultiEdit|Agent|Task`)
- Token logging appends to `.planning/token-log.jsonl` (same file as all other hooks)

</code_context>

<deferred>
## Deferred Ideas

- Full-file analysis mode (send entire file instead of diff for deeper review) — evaluate if diff-only misses too many issues
- Configurable scan categories (bugs-only, security-only, performance) via settings

</deferred>

---

*Phase: 11-posttooluse-bug-scanner*
*Context gathered: 2026-04-03*
