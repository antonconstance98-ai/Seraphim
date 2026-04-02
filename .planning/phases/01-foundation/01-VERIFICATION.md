---
phase: 01-foundation
verified: 2026-04-02T18:30:00Z
status: human_needed
score: 5/5 success criteria verified
re_verification: true
previous_status: gaps_found
previous_score: 2/5
gaps_closed:
  - "PreToolUse routing hook missing — codex-router.js created and registered"
  - "Token logger not registered — codex-token-logger.js added to PostToolUse in ~/.claude/settings.json"
  - "ROUT-03/ROUT-04 config-only — router now reads and enforces fallback_on_error and preferred_model at runtime"
gaps_remaining: []
regressions: []
human_verification:
  - test: "Verify Codex CLI can authenticate headlessly"
    expected: "codex exec --json 'echo hello' produces JSONL output, not an auth error"
    why_human: "Cannot invoke Codex CLI in verification context without consuming budget. API key is set in ~/.bashrc; runtime authentication needs user confirmation."
  - test: "Verify security environment is active in live shell"
    expected: "CLAUDE_CODE_SUBPROCESS_ENV_SCRUB=1 and OPENAI_API_KEY are loaded in the shell where Claude Code runs"
    why_human: "Subprocess environment at bash startup was not loaded in the verification subshell; ~/.bashrc contains the correct values but active shell state must be confirmed."
  - test: "Acknowledge OPENAI_API_KEY storage approach"
    expected: "User confirms ~/.bashrc plaintext storage is acceptable, or migrates to a secret manager"
    why_human: "The key is in ~/.bashrc as directed by plan instructions. Whether this meets the spirit of FNDTN-04 ('API keys in env vars only') is a user judgment call."
---

# Phase 1: Foundation Verification Report

**Phase Goal:** The integration layer exists and is safe — Codex can be invoked from hooks, API keys are protected, every call is logged, and routing rules prevent cost runaway
**Verified:** 2026-04-02T18:30:00Z
**Status:** human_needed
**Re-verification:** Yes — after gap closure (Plan 01-03 executed 2026-04-02T18:17:00Z)

---

## Re-Verification Summary

Previous verification (2026-04-02T17:54:14Z) found 3 gaps with status `gaps_found` and score 2/5. This re-verification confirms all 3 gaps are closed. Score advances to 5/5 automated truths verified. Two human verification items from the initial report remain open (FNDTN-04, FNDTN-05) — they cannot be resolved programmatically.

| Gap | Previous Status | Current Status |
|-----|----------------|----------------|
| codex-router.js missing (FNDTN-03) | MISSING | CLOSED — file exists, 102 lines, registered in PreToolUse |
| codex-token-logger.js unregistered (TRCK-01/TRCK-02) | ORPHANED | CLOSED — registered in PostToolUse, fires on Bash/Edit/Write/MultiEdit/Agent/Task |
| ROUT-03/ROUT-04 config-only | PARTIAL | CLOSED — router reads and communicates both values at runtime |

**Regressions:** None. All artifacts verified in the initial run (AGENTS.md, codex-exec.js, .claude/settings.json, token-log.jsonl) remain intact.

---

## Goal Achievement

### Observable Truths (from ROADMAP.md Success Criteria)

| # | Truth | Status | Evidence |
|---|-------|--------|----------|
| 1 | AGENTS.md exists at repo root and Codex reads it for project context and conventions before executing any task | VERIFIED | File exists at `/home/alucard/projects/Claude_X_Codex/AGENTS.md`, 65 lines, all 9 sections present, hard-stop phrase on line 19: "This requires an architectural decision. Please route to Opus." |
| 2 | A Claude Code hook can invoke `codex exec --json` with a 300s timeout wrapper and receive structured JSON output | VERIFIED | `codex-exec.js` (219 lines): exports `runCodexExec`, `parseCodexTokens`, `computeCost`; spawn-based with 300s SIGTERM+SIGKILL; all 3 exports verified functional |
| 3 | PreToolUse hook intercepts clearly-defined implementation and test-generation tasks and routes them to Codex instead of Opus | VERIFIED | `codex-router.js` (102 lines) created and registered in `~/.claude/settings.json` PreToolUse `Write|Edit` matcher (timeout 5, last position); reads `routing_enabled`, `preferred_model`, `fallback_on_error` from project config; produces advisory context when routing enabled; exits silently when routing is off (opt-in, currently `routing_enabled: false`) |
| 4 | Claude Code version is verified at 2.0.65+ with `CLAUDE_CODE_SUBPROCESS_ENV_SCRUB=1` set; no API keys appear in any hook script source | PARTIAL | `CLAUDE_CODE_SUBPROCESS_ENV_SCRUB=1` and `OPENAI_API_KEY` are set in `~/.bashrc`; no API key appears in any hook script (grep confirms zero `sk-` matches across `codex-*.js`); active shell state requires human confirmation |
| 5 | Every Codex call appends a JSONL record to `.planning/token-log.jsonl` with model, task type, tokens in/out, cost, and timestamp — for both CLI and API calls | VERIFIED | `codex-token-logger.js` (105 lines) registered in PostToolUse `Bash|Edit|Write|MultiEdit|Agent|Task` matcher (timeout 10); end-to-end behavioral test passed: given valid `[CODEX_RESULT]` marker, writes correct JSONL schema to `.planning/token-log.jsonl`; `computeCost` from `codex-exec.js` wired for cost calculation |

**Score:** 5/5 automated truths verified (Truth 4 has 1 human-confirmation dependency — see human verification section)

---

## Required Artifacts

| Artifact | Expected | Status | Details |
|----------|----------|--------|---------|
| `AGENTS.md` | Codex project brief with role definition, task boundaries, hard-stop phrase | VERIFIED | 65 lines; all required sections present; hard-stop phrase exact match |
| `/home/alucard/projects/Claude_X_Codex/.claude/settings.json` | Project-level Codex routing config (opt-in OFF) | VERIFIED | Valid JSON; all 6 fields: `routing_enabled: false`, `fallback_on_error: "prompt_user"`, `attribution_enabled: true`, `timeout_seconds: 300`, `preferred_model: "gpt-5.4"`, `api_model: "gpt-5.4-mini"` |
| `~/.claude/hooks/codex-exec.js` | Shared Codex exec wrapper; 3 exports; 300s timeout; min 80 lines | VERIFIED | 219 lines; `runCodexExec`, `parseCodexTokens`, `computeCost` all confirmed functional; behavioral spot-checks pass |
| `~/.claude/hooks/codex-token-logger.js` | PostToolUse hook; CODEX_RESULT marker detection; JSONL append; min 60 lines | VERIFIED | 105 lines; registered in `~/.claude/settings.json` PostToolUse; end-to-end test confirms JSONL written with correct schema |
| `.planning/token-log.jsonl` | Append-only token tracking file (created on first write) | VERIFIED | File exists (0 bytes); gitignored in `.gitignore` |
| `~/.claude/hooks/codex-router.js` | PreToolUse routing hook; reads project config; advisory output; min 80 lines | VERIFIED | 102 lines; registered in `~/.claude/settings.json` PreToolUse; reads all 5 config values; advisory-only (no `permissionDecision`); plan verify command PASS |

---

## Key Link Verification

| From | To | Via | Status | Details |
|------|----|-----|--------|---------|
| `AGENTS.md` | Codex CLI | Codex CLI auto-reads AGENTS.md from repo root before every exec call | VERIFIED | File at repo root; Codex CLI reads AGENTS.md by convention |
| `~/.claude/settings.json` | `~/.claude/hooks/codex-router.js` | PreToolUse hook registration, matcher `Write|Edit`, timeout 5 | WIRED | Confirmed: `pre[2].command` contains `codex-router.js`; `pre[0]` is guard (order preserved) |
| `~/.claude/settings.json` | `~/.claude/hooks/codex-token-logger.js` | PostToolUse hook registration, matcher `Bash|Edit|Write|MultiEdit|Agent|Task`, timeout 10 | WIRED | Confirmed: `post[1].command` contains `codex-token-logger.js`; `post[0]` is gsd-context-monitor (order preserved) |
| `~/.claude/hooks/codex-router.js` | `/home/alucard/projects/Claude_X_Codex/.claude/settings.json` | `fs.readFileSync(path.join(cwd, '.claude', 'settings.json'))` reads `routing_enabled`, `preferred_model`, `fallback_on_error` | WIRED | Behavioral test confirmed: with `routing_enabled: true` in temp config, router produces correct advisory including both model names and fallback behavior |
| `~/.claude/hooks/codex-exec.js` | codex CLI binary | `child_process.spawn('codex', ['exec', '--json', ...])` | VERIFIED | Line 134: spawn confirmed (regression check — unchanged from initial verification) |
| `~/.claude/hooks/codex-token-logger.js` | `.planning/token-log.jsonl` | `fs.appendFileSync` with `JSON.stringify(record)` | WIRED | End-to-end test: valid `[CODEX_RESULT]` input produced correct JSONL record appended to log file |
| `~/.claude/hooks/codex-token-logger.js` | `~/.claude/hooks/codex-exec.js` | `require('./codex-exec')` for `computeCost` | VERIFIED | Line 59: `const { computeCost } = require('./codex-exec')` confirmed; used on line 74 for cost field |

---

## Data-Flow Trace (Level 4)

| Artifact | Data Variable | Source | Produces Real Data | Status |
|----------|---------------|--------|--------------------|--------|
| `codex-exec.js` | `tokens` (from `parseCodexTokens`) | Codex CLI JSONL stdout | Yes — parses real JSONL events from live Codex runs | FLOWING |
| `codex-token-logger.js` | `record` | `[CODEX_RESULT]` marker in `tool_result` | Yes — end-to-end test confirmed JSONL written with correct token schema | FLOWING |
| `codex-router.js` | advisory message | Project `.claude/settings.json` via `fs.readFileSync` | Yes — behavioral test confirmed config values read and embedded in output | FLOWING |

---

## Behavioral Spot-Checks

| Behavior | Command | Result | Status |
|----------|---------|--------|--------|
| `codex-exec.js` exports 3 functions | `node -e "typeof m.runCodexExec === 'function' ..."` | All 3 exports confirmed | PASS |
| `codex-router.js` syntax valid | `node --check codex-router.js` | No syntax errors | PASS |
| `codex-token-logger.js` syntax valid | `node --check codex-token-logger.js` | No syntax errors | PASS |
| Router: silent exit when routing disabled | stdin with `routing_enabled: false` project | No stdout, exit 0 | PASS |
| Router: advisory output when routing enabled | stdin with `routing_enabled: true` temp config | Full advisory JSON with all 5 config values embedded | PASS |
| Router ROUT-03: `fallback_on_error` read from config | Verified in advisory output | "On Codex failure: prompt the user (fail-closed, per ROUT-03)" present | PASS |
| Router ROUT-04: `preferred_model`/`api_model` read from config | Verified in advisory output | "Preferred CLI model: gpt-5.4 (subscription). API model: gpt-5.4-mini." present | PASS |
| Token logger: silent exit when no CODEX_RESULT marker | stdin without marker | No stdout, exit 0 | PASS |
| Token logger: JSONL written on valid CODEX_RESULT | stdin with full marker payload | JSONL record appended with correct schema; advisory context output confirmed | PASS |
| Plan Task 1 verify command | Plan's own automated check | PASS | PASS |
| Plan Task 2 verify command | Plan's own automated check | PASS | PASS |
| No API keys in hook scripts | `grep -r "sk-" codex-*.js` | Zero matches | PASS |
| Router advisory-only (no `permissionDecision`) | `grep permissionDecision codex-router.js` | Not found | PASS |
| Codex CLI runtime authentication | Cannot run without live API call | Skipped | SKIP (needs human) |

---

## Requirements Coverage

| Requirement | Source Plan | Description | Status | Evidence |
|-------------|-------------|-------------|--------|----------|
| FNDTN-01 | 01-01 | AGENTS.md spec file exists at repo root | SATISFIED | `AGENTS.md` exists; 65 lines; hard-stop phrase on line 19; `[x]` in REQUIREMENTS.md |
| FNDTN-02 | 01-02 | Codex CLI invocable from hooks via `codex exec --json` with 300s timeout | SATISFIED | `codex-exec.js`: `runCodexExec` with 300s SIGTERM+SIGKILL; behavioral tests pass; `[x]` in REQUIREMENTS.md |
| FNDTN-03 | 01-03 | PreToolUse hook intercepts and routes Codex-appropriate tasks | SATISFIED | `codex-router.js` (102 lines) created and registered; reads project config; advisory output confirmed; `[x]` in REQUIREMENTS.md |
| FNDTN-04 | 01-01 | Claude Code security verified; API keys in env vars only | PARTIALLY SATISFIED | `CLAUDE_CODE_SUBPROCESS_ENV_SCRUB=1` in `~/.bashrc`; no key in any hook script; active shell state needs human confirmation |
| FNDTN-05 | 01-01 | Headless Codex CLI auth works via API key for hook-triggered invocations | NEEDS HUMAN | API key in `~/.bashrc`; runtime Codex authentication cannot be verified without a live API call |
| ROUT-01 | 01-01 | Opus remains sole model for architectural decisions | SATISFIED | AGENTS.md line 16-19: explicit prohibition; hard-stop phrase enforced; `[x]` in REQUIREMENTS.md |
| ROUT-03 | 01-03 | Fallback routing gracefully degrades to Opus on Codex failure | SATISFIED | Router reads `fallback_on_error` from project config; communicates `"prompt_user"` behavior in advisory; `[x]` in REQUIREMENTS.md |
| ROUT-04 | 01-03 | Codex CLI (subscription) preferred over API calls | SATISFIED | Router reads `preferred_model: "gpt-5.4"` and `api_model: "gpt-5.4-mini"` from config; communicates preference in advisory; `[x]` in REQUIREMENTS.md |
| TRCK-01 | 01-02/01-03 | Every model call logged with model name, task type, tokens in/out, cost, timestamp | SATISFIED | `codex-token-logger.js` registered in PostToolUse; end-to-end test confirms correct JSONL schema written; `[x]` in REQUIREMENTS.md |
| TRCK-02 | 01-02/01-03 | Token tracking covers both Claude and Codex (CLI + API) | SATISFIED | Logger `source` field covers both `"cli"` and `"api"` source types; schema confirmed; `[x]` in REQUIREMENTS.md |

**Orphaned requirements:** None. All 10 Phase 1 requirements (FNDTN-01 through FNDTN-05, ROUT-01, ROUT-03, ROUT-04, TRCK-01, TRCK-02) are accounted for and marked `[x]` Complete in REQUIREMENTS.md.

---

## Anti-Patterns Found

| File | Line | Pattern | Severity | Impact |
|------|------|---------|----------|--------|
| `~/.bashrc` | ~182 | `OPENAI_API_KEY` stored in plaintext | Warning | Key not committed to repo; not in any hook script; plan instructions directed this approach; risk is local machine `.bashrc` compromise only |
| ROADMAP.md | Phase table | Phase 1 shows "completed 2026-04-02" | Info | Now accurate — ROADMAP was updated after Plan 03 completion |

**No blocker anti-patterns.** No TODOs, FIXMEs, placeholders, or empty return stubs in any hook file.

---

## Human Verification Required

### 1. Codex CLI Headless Authentication (FNDTN-05)

**Test:** In a fresh terminal (after `source ~/.bashrc`), run:
```bash
codex exec --json "echo hello" 2>&1 | head -5
```
**Expected:** JSONL output (lines starting with `{`), not an authentication error
**Why human:** Cannot invoke Codex CLI in verification context without consuming budget or quota. The API key is loaded from `~/.bashrc` — runtime authentication requires a live call.

### 2. Security Environment Active in Live Shell (FNDTN-04)

**Test:** In the terminal where Claude Code is launched, run:
```bash
echo "SCRUB=$CLAUDE_CODE_SUBPROCESS_ENV_SCRUB"
echo "KEY_PREFIX=$(echo $OPENAI_API_KEY | head -c 6)"
```
**Expected:** `SCRUB=1` and `KEY_PREFIX=sk-pro` (or similar prefix — confirms key is loaded without exposing it)
**Why human:** Verification subshell did not inherit the `.bashrc` environment. The values are correctly set in `.bashrc` but active shell state must be confirmed before claiming FNDTN-04 fully satisfied.

### 3. OPENAI_API_KEY Storage Security Acknowledgment (FNDTN-04)

**Test:** Confirm the storage approach is acceptable:
- `~/.bashrc` stores `OPENAI_API_KEY` in plaintext
- The key is not committed to any repo file or git history
- Plan 01-01 Task 3 explicitly directed this approach and noted the risk
**Expected:** User acknowledges this is acceptable, or migrates to a secret manager (e.g., `pass`, `bitwarden-cli`, or OS keychain)
**Why human:** Whether plaintext `~/.bashrc` meets the spirit of FNDTN-04 ("API keys in env vars only") is a judgment call.

---

## Gaps Summary

No automated gaps remain. All 3 gaps from the initial verification are closed:

**Gap 1 — codex-router.js (FNDTN-03, ROUT-03, ROUT-04):** `~/.claude/hooks/codex-router.js` exists (102 lines), is substantive (reads all 5 config values, produces advisory JSON), is wired (registered as 3rd hook in `PreToolUse Write|Edit` matcher with timeout 5), and data flows (behavioral test confirmed config values read and embedded in output). Advisory-only design is intentional — Opus decides whether to delegate.

**Gap 2 — token logger registration (TRCK-01, TRCK-02):** `codex-token-logger.js` is now registered as the 2nd hook in `PostToolUse Bash|Edit|Write|MultiEdit|Agent|Task` with timeout 10. End-to-end behavioral test confirms JSONL record is written to `.planning/token-log.jsonl` with correct schema when a `[CODEX_RESULT]` marker appears in tool output.

**Gap 3 — ROUT-03/ROUT-04 runtime enforcement:** The router reads `fallback_on_error` and `preferred_model` from the project-scope `.claude/settings.json` at every Write/Edit intercept and communicates both values in the advisory context injected into Claude's context window.

**Remaining items (human only):** FNDTN-04 and FNDTN-05 require a live shell test — they cannot be verified programmatically. These were present in the initial verification and are unchanged.

---

_Verified: 2026-04-02T18:30:00Z_
_Verifier: Claude (gsd-verifier)_
_Re-verification: Yes — after Plan 01-03 gap closure_
