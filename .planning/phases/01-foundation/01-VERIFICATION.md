---
phase: 01-foundation
verified: 2026-04-02T17:54:14Z
status: gaps_found
score: 3/5 success criteria verified
re_verification: false
gaps:
  - truth: "PreToolUse hook intercepts clearly-defined implementation and test-generation tasks and routes them to Codex instead of Opus"
    status: failed
    reason: "codex-router.js does not exist anywhere in ~/.claude/hooks/. FNDTN-03 is marked Pending in REQUIREMENTS.md. No PreToolUse hook for Codex routing was built in Phase 1."
    artifacts:
      - path: "~/.claude/hooks/codex-router.js"
        issue: "File does not exist — MISSING"
    missing:
      - "Create ~/.claude/hooks/codex-router.js PreToolUse hook that reads .claude/settings.json routing_enabled and intercepts implementation/test-gen tasks"
      - "Register codex-router.js in ~/.claude/settings.json PreToolUse hooks"

  - truth: "Every Codex call appends a JSONL record to .planning/token-log.jsonl with model, task type, tokens in/out, cost, and timestamp"
    status: partial
    reason: "codex-token-logger.js exists and is fully implemented but is NOT registered in ~/.claude/settings.json. The plan explicitly deferred hook registration to Plan 03, meaning token logging cannot fire in any live session. The artifact is ORPHANED — substantive but unwired."
    artifacts:
      - path: "~/.claude/hooks/codex-token-logger.js"
        issue: "Not registered in ~/.claude/settings.json PostToolUse hooks — script exists but is never invoked"
    missing:
      - "Register codex-token-logger.js in ~/.claude/settings.json PostToolUse section with appropriate matcher"

  - truth: "ROUT-03 and ROUT-04: Fallback routing degrades gracefully; Codex CLI (subscription) preferred over API"
    status: partial
    reason: "Config values exist (fallback_on_error: prompt_user, preferred_model: gpt-5.4, api_model: gpt-5.4-mini) but the behavioral requirement — actual runtime routing logic that enforces the preference and handles fallback — is not implemented. REQUIREMENTS.md correctly marks both as Pending. Config documents intent; hooks implement it."
    artifacts:
      - path: ".claude/settings.json"
        issue: "Contains config values for ROUT-03/ROUT-04 but no hook reads these values at runtime — the routing logic that would enforce them does not exist"
    missing:
      - "codex-router.js must read preferred_model and fallback_on_error from .claude/settings.json and enforce them at routing time"

human_verification:
  - test: "Verify OPENAI_API_KEY is safely stored"
    expected: "Key exists only in ~/.bashrc (not in any project file, not in git history); CLAUDE_CODE_SUBPROCESS_ENV_SCRUB=1 is active in the running shell"
    why_human: "~/.bashrc contains the key in plaintext — this matches the plan instructions ('Add to ~/.bashrc') but differs from the FNDTN-05 requirement spirit ('API keys in env vars only'). The key is not in any committed file, but storing it in .bashrc plaintext is a security trade-off the user should acknowledge."
  - test: "Verify Codex CLI can authenticate headlessly"
    expected: "codex exec --json 'echo hello' produces JSONL output, not an auth error"
    why_human: "Cannot invoke Codex CLI in this verification context without consuming budget. The API key is set in .bashrc; runtime authentication needs user confirmation."
---

# Phase 1: Foundation Verification Report

**Phase Goal:** The integration layer exists and is safe — Codex can be invoked from hooks, API keys are protected, every call is logged, and routing rules prevent cost runaway
**Verified:** 2026-04-02T17:54:14Z
**Status:** gaps_found
**Re-verification:** No — initial verification

---

## Goal Achievement

### Observable Truths (from ROADMAP.md Success Criteria)

| # | Truth | Status | Evidence |
|---|-------|--------|----------|
| 1 | AGENTS.md exists at repo root and Codex reads it for project context before executing any task | VERIFIED | File exists at `/home/alucard/projects/Claude_X_Codex/AGENTS.md`; contains all 9 required sections; hard-stop phrase present; auto-read by Codex CLI by convention |
| 2 | A Claude Code hook can invoke `codex exec --json` with a 300s timeout wrapper and receive structured JSON output | VERIFIED | `codex-exec.js` exports `runCodexExec`, `parseCodexTokens`, `computeCost`; all functional tests pass; spawn-based with SIGTERM+SIGKILL at 300s |
| 3 | PreToolUse hook intercepts clearly-defined implementation and test-generation tasks and routes them to Codex instead of Opus | FAILED | `codex-router.js` does not exist; no PreToolUse Codex routing hook is registered; FNDTN-03 marked Pending in REQUIREMENTS.md |
| 4 | Claude Code version is verified at 2.0.65+ with `CLAUDE_CODE_SUBPROCESS_ENV_SCRUB=1` set; no API keys appear in any hook script source | PARTIAL | `CLAUDE_CODE_SUBPROCESS_ENV_SCRUB=1` is set in `~/.bashrc`; `OPENAI_API_KEY` is set in `~/.bashrc` (plaintext but not committed to repo); no API key appears in any hook script; runtime env needs human confirmation |
| 5 | Every Codex call appends a JSONL record to `.planning/token-log.jsonl` with model, task type, tokens in/out, cost, and timestamp — for both CLI and API calls | FAILED | `codex-token-logger.js` is fully implemented and correct but is NOT registered in `~/.claude/settings.json` — it will never fire; hook registration explicitly deferred to Plan 03 |

**Score:** 2/5 truths fully verified (1 partial, 2 failed)

---

## Required Artifacts

| Artifact | Expected | Status | Details |
|----------|----------|--------|---------|
| `AGENTS.md` | Codex project brief with role definition, task boundaries, tech stack, conventions, security rules | VERIFIED | 65 lines; all 9 sections present; hard-stop phrase exact |
| `.claude/settings.json` | Project-level Codex routing config (opt-in OFF) | VERIFIED | Valid JSON; all 6 fields correct; `routing_enabled: false` |
| `~/.claude/hooks/codex-exec.js` | Shared Codex exec wrapper; exports `runCodexExec`, `parseCodexTokens`, `computeCost`; min 80 lines | VERIFIED | 219 lines; all 3 exports present and functional; 300s timeout with SIGTERM+SIGKILL; no API key leakage |
| `~/.claude/hooks/codex-token-logger.js` | PostToolUse hook; CODEX_RESULT marker detection; JSONL append; min 60 lines | ORPHANED | 105 lines; fully implemented; NOT registered in `~/.claude/settings.json` — will never fire |
| `.planning/token-log.jsonl` | Append-only token tracking file (created on first write) | VERIFIED | File exists (0 bytes); gitignored in `.gitignore` |
| `~/.claude/hooks/codex-router.js` | PreToolUse routing hook (implied by key_link in 01-01) | MISSING | Does not exist |

---

## Key Link Verification

| From | To | Via | Status | Details |
|------|----|-----|--------|---------|
| `AGENTS.md` | Codex CLI | Codex CLI auto-reads AGENTS.md from repo root before every exec call | VERIFIED | File exists at repo root; Codex CLI behavior is standard — no code required |
| `.claude/settings.json` | `~/.claude/hooks/codex-router.js` | Routing hook reads project config to check opt-in flag | NOT_WIRED | `codex-router.js` does not exist; link cannot be established |
| `~/.claude/hooks/codex-exec.js` | codex CLI binary | `child_process.spawn('codex', ['exec', '--json', ...])` | VERIFIED | Line 134: `spawn('codex', args, ...)` confirmed |
| `~/.claude/hooks/codex-token-logger.js` | `.planning/token-log.jsonl` | `fs.appendFileSync` with `JSON.stringify(record)` | PARTIAL | Code is correct (line 89); but hook is not registered in `~/.claude/settings.json` so the append never executes |
| `~/.claude/hooks/codex-token-logger.js` | `~/.claude/hooks/codex-exec.js` | `require('./codex-exec')` for `computeCost` | VERIFIED | Line 59: `const { computeCost } = require('./codex-exec')` confirmed |

---

## Data-Flow Trace (Level 4)

| Artifact | Data Variable | Source | Produces Real Data | Status |
|----------|---------------|--------|--------------------|--------|
| `codex-exec.js` | `tokens` (from `parseCodexTokens`) | Codex CLI JSONL stdout | Yes — parses real JSONL events from live Codex runs | FLOWING |
| `codex-token-logger.js` | `record` | `[CODEX_RESULT]` marker in `tool_result` | Would flow — but hook never fires (unregistered) | HOLLOW — hook unregistered |

---

## Behavioral Spot-Checks

| Behavior | Command | Result | Status |
|----------|---------|--------|--------|
| `codex-exec.js` exports 3 functions | `node -e "... typeof m.runCodexExec === 'function' ..."` | All 3 exports confirmed | PASS |
| `parseCodexTokens` parses real JSONL schema | Verified JSONL event with known values | Returns correct token counts | PASS |
| `parseCodexTokens` returns null for empty input | Empty string input | `null` returned | PASS |
| `parseCodexTokens` uses LAST non-null token_count event (Pitfall 3) | Two-event JSONL — first null, second with data | Returns second event values | PASS |
| `computeCost` returns correct value for gpt-5.4 | 1000 input + 100 output tokens | 0.0035 USD (expected) | PASS |
| `computeCost` returns correct value for gpt-5.4-mini | 1000 input + 500 cached + 100 output tokens | 0.00066 USD (expected) | PASS |
| `codex-token-logger.js` syntax check | `node -c codex-token-logger.js` | No syntax errors | PASS |
| Codex CLI runtime authentication | Cannot run without live API call | Skipped | SKIP (needs human) |

---

## Requirements Coverage

| Requirement | Source Plan | Description | Status | Evidence |
|-------------|-------------|-------------|--------|----------|
| FNDTN-01 | 01-01 | AGENTS.md spec file exists at repo root | SATISFIED | `AGENTS.md` exists with all required sections; hard-stop phrase present |
| FNDTN-02 | 01-02 | Codex CLI invocable from hooks via `codex exec --json` with 300s timeout | SATISFIED | `codex-exec.js` runCodexExec: spawn + 300s SIGTERM+SIGKILL; functional tests pass |
| FNDTN-03 | (Phase 1 ROADMAP) | PreToolUse hook intercepts and routes Codex-appropriate tasks | BLOCKED | No routing hook exists; `codex-router.js` missing; REQUIREMENTS.md marks Pending |
| FNDTN-04 | 01-01 | Claude Code security verified; API keys in env vars only | PARTIALLY SATISFIED | `CLAUDE_CODE_SUBPROCESS_ENV_SCRUB=1` in `.bashrc`; `OPENAI_API_KEY` in `.bashrc` (plaintext, not committed); needs user confirmation of active shell state |
| FNDTN-05 | 01-01 | Headless Codex CLI auth works via API key for hook-triggered invocations | NEEDS HUMAN | API key is set in `.bashrc`; runtime Codex authentication cannot be verified without a live API call |
| ROUT-01 | 01-01 | Opus remains sole model for architectural decisions — enforced by AGENTS.md | SATISFIED | AGENTS.md contains the hard-stop phrase and "What You Do NOT Handle" section enforcing this rule |
| ROUT-03 | 01-01 | Fallback routing degrades gracefully to Opus on Codex failure | BLOCKED | Config value `fallback_on_error: "prompt_user"` exists but no runtime routing hook reads it; behavioral requirement not implemented |
| ROUT-04 | 01-01 | Codex CLI (subscription) preferred over API calls | BLOCKED | Config values `preferred_model`/`api_model` exist but no routing hook enforces the preference at runtime |
| TRCK-01 | 01-02 | Every model call logged with model name, task type, tokens in/out, cost, timestamp | BLOCKED | `codex-token-logger.js` implements the correct schema but is NOT registered in `~/.claude/settings.json` — logging never fires |
| TRCK-02 | 01-02 | Token tracking covers both Claude and Codex (CLI + API) | BLOCKED | Schema covers both `cli` and `api` source types; implementation correct; same blocker as TRCK-01 — unregistered hook |

**Orphaned requirements for Phase 1 not in any plan's `requirements` field:** None — all 10 Phase 1 requirements are accounted for across the two plans.

**Discrepancy noted:** ROUT-03 and ROUT-04 appear in Plan 01-01's `requirements` array but are marked Pending in `REQUIREMENTS.md` and have no implementation. The plan's config values document intent but do not constitute behavioral implementation of these requirements.

---

## Anti-Patterns Found

| File | Line | Pattern | Severity | Impact |
|------|------|---------|----------|--------|
| `~/.bashrc` | 182 | `OPENAI_API_KEY` stored in plaintext | Warning | Key is not committed to repo and not exposed in any hook script; risk is local machine compromise of `.bashrc`; plan instructions explicitly directed this approach; plan 01-01 Task 3 explicitly states "Never commit ~/.bashrc or share your API key" |
| `~/.claude/hooks/codex-token-logger.js` | — | Fully implemented hook not registered in settings | Warning | Token logging silently never fires; no runtime error occurs; any Phase 2 work that depends on token data will find an empty log |
| ROADMAP.md | 74 | Phase 1 shows "In Progress" but STATE.md claims `completed_phases: 1` | Info | STATE.md was updated prematurely; ROADMAP table was not updated after Plan 02 completion |

---

## Human Verification Required

### 1. Codex CLI Headless Authentication (FNDTN-05)

**Test:** In a fresh terminal (after `source ~/.bashrc`), run:
```bash
codex exec --json "echo hello" 2>&1 | head -5
```
**Expected:** JSONL output (lines starting with `{`), not an authentication error
**Why human:** Cannot invoke Codex CLI in verification context without consuming budget/quota

### 2. Security Environment Active in Live Shell (FNDTN-04)

**Test:** In the terminal where Claude Code is launched, run:
```bash
echo "SCRUB=$CLAUDE_CODE_SUBPROCESS_ENV_SCRUB"
echo "KEY_PREFIX=$(echo $OPENAI_API_KEY | head -c 6)"
```
**Expected:** `SCRUB=1` and `KEY_PREFIX=sk-pro` (or similar — confirms key is loaded without exposing it)
**Why human:** Subprocess environment at bash startup was not loaded in the verification subshell; `.bashrc` contains the correct values but active shell state must be confirmed

### 3. OPENAI_API_KEY Storage Security Acknowledgment

**Test:** Confirm the storage approach is acceptable:
- `~/.bashrc` line 181-182 stores both env vars in plaintext
- The key is `sk-proj-0cO4iNMpm...` (real key stored in home directory config file)
**Expected:** User acknowledges this is acceptable or migrates to a secret manager (e.g., `pass`, `bitwarden-cli`, or OS keychain)
**Why human:** The plan instructions specified `.bashrc` as the storage location and acknowledged the risk. Whether this meets the spirit of "API keys in env vars only" (FNDTN-04) is a judgment call.

---

## Gaps Summary

Phase 1 has 3 blocking gaps preventing full goal achievement:

**Gap 1 — PreToolUse routing hook missing (FNDTN-03, Success Criterion 3):** The most significant structural gap. No `codex-router.js` exists. The phase goal states "routing rules prevent cost runaway" — without a routing hook, Codex is never invoked from hooks at all. The config opt-in flag (`routing_enabled`) exists but nothing reads it. This gap means the core routing capability of Phase 1 is absent.

**Gap 2 — Token logger not registered (TRCK-01, TRCK-02, Success Criterion 5):** `codex-token-logger.js` is a correct, substantive implementation that will never execute. The plan text says "Do NOT register this hook yet — that happens in Plan 03." This means Plan 03 (which doesn't exist in the phase directory) was supposed to complete Phase 1 by registering hooks. Phase 1 was marked complete in STATE.md with only 2 of 3 planned plans, leaving token logging permanently unregistered unless corrected.

**Gap 3 — ROUT-03/ROUT-04 config-only (not behavioral):** The config values `fallback_on_error: "prompt_user"` and `preferred_model: "gpt-5.4"` document intent but no hook enforces them. These require Gap 1 (codex-router.js) to be resolved first — the router hook would read these values and act on them.

**Root cause:** Gaps 1, 2, and 3 share a single root cause — Plan 03 (hook registration + routing hook creation) was planned but never executed. STATE.md was updated to show Phase 1 complete after only 2 of 3 plans.

---

_Verified: 2026-04-02T17:54:14Z_
_Verifier: Claude (gsd-verifier)_
