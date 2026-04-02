---
phase: 01-foundation
plan: 01
subsystem: infra
tags: [codex, agents, hooks, security, openai, settings]

requires: []
provides:
  - AGENTS.md Codex project brief at repo root (auto-read by Codex CLI before every task)
  - Project-scope .claude/settings.json with Codex routing config (opt-in OFF by default)
affects: [02-hooks-infrastructure, 03-plan-review-loop-superpowers, 04-token-tracking]

tech-stack:
  added: []
  patterns:
    - "AGENTS.md at repo root: Codex reads this automatically before every exec call"
    - "Project-scope .claude/settings.json: Codex routing config separate from user-scope ~/.claude/settings.json"
    - "Codex routing opt-in OFF by default: set routing_enabled=true per-project to activate"

key-files:
  created:
    - AGENTS.md
    - .claude/settings.json
  modified: []

key-decisions:
  - "D-01 moderate guardrails: Codex handles minor judgment calls; must escalate structural decisions to Opus"
  - "D-07 opt-in routing: routing_enabled=false by default; user explicitly enables per project"
  - "D-08 attribution: attribution_enabled=true so user always knows which model did the work"
  - "ROUT-03 fail-closed: fallback_on_error=prompt_user, not silent auto-route to Opus"

patterns-established:
  - "AGENTS.md living document: updated at each GSD phase transition with new conventions"
  - "Hard-stop phrase: Codex responds 'This requires an architectural decision. Please route to Opus.' for structural decisions"

requirements-completed:
  - FNDTN-01
  - FNDTN-04
  - FNDTN-05
  - ROUT-01

duration: 2min
completed: 2026-04-02
---

# Phase 01 Plan 01: Foundation Security Baseline and Codex Project Brief Summary

**AGENTS.md Codex project brief and project-scope .claude/settings.json with routing opt-in OFF, security env vars pending user action**

## Performance

- **Duration:** ~2 min
- **Started:** 2026-04-02T17:26:51Z
- **Completed:** 2026-04-02T17:28:19Z
- **Tasks:** 2 of 3 complete (Task 3 is a human-action checkpoint)
- **Files modified:** 2

## Accomplishments

- Created AGENTS.md at repo root with all 9 required sections: role definition, task boundaries, tech stack, conventions, security rules, communication style, and the hard-stop phrase for architectural decisions
- Created .claude/settings.json with Codex routing config (routing OFF by default, fail-closed fallback, attribution enabled, 300s timeout, correct model names)
- Both files verified correct via automated checks

## Task Commits

Each task was committed atomically:

1. **Task 1: Create AGENTS.md Codex project brief** - `d6fb659` (feat)
2. **Task 2: Create project-scope .claude/settings.json with Codex opt-in config** - `d0f3386` (chore)
3. **Task 3: Set security environment variables** - PENDING (checkpoint:human-action)

## Files Created/Modified

- `/home/alucard/projects/Claude_X_Codex/AGENTS.md` - Codex project brief: role, task boundaries, tech stack, conventions, security rules, communication style
- `/home/alucard/projects/Claude_X_Codex/.claude/settings.json` - Project-scope Codex routing config with all 6 config fields

## Decisions Made

- D-07 routing opt-in: routing_enabled=false by default. User must explicitly set to true to activate Codex routing in this project.
- D-08 attribution: attribution_enabled=true so the user always knows when Codex handled something.
- ROUT-03 fail-closed: fallback_on_error="prompt_user" rather than silently auto-routing to Opus on failure.
- Hard-stop phrase exact wording: "This requires an architectural decision. Please route to Opus." — Codex must use this exact phrase, enabling automated detection in hook scripts.

## Deviations from Plan

None - plan executed exactly as written.

## Issues Encountered

None.

## User Setup Required

Task 3 (checkpoint:human-action) requires two manual steps before any hooks can safely invoke Codex:

**Step 1: Set CLAUDE_CODE_SUBPROCESS_ENV_SCRUB=1**
This prevents API key exfiltration via hook scripts (CVE-2025-59536). Run:
```bash
echo 'export CLAUDE_CODE_SUBPROCESS_ENV_SCRUB=1' >> ~/.bashrc
source ~/.bashrc
```
Verify: `echo $CLAUDE_CODE_SUBPROCESS_ENV_SCRUB` should print `1`

**Step 2: Set OPENAI_API_KEY**
Required for hook-triggered Codex invocations. Get your key from https://platform.openai.com/api-keys then:
```bash
echo 'export OPENAI_API_KEY="sk-your-key-here"' >> ~/.bashrc
source ~/.bashrc
```
Verify: `echo $OPENAI_API_KEY` should print your key (starts with "sk-")

**Step 3: Verify Codex authentication**
```bash
OPENAI_API_KEY="$OPENAI_API_KEY" codex exec --json "echo hello" 2>&1 | head -5
```
Should produce JSONL output (not an auth error).

**Important:** Never commit ~/.bashrc or share your API key. The key lives only in your shell environment.

## Next Phase Readiness

- AGENTS.md and .claude/settings.json are ready for Phase 02 hooks infrastructure to reference
- Phase 02 routing hook will read .claude/settings.json to check routing_enabled flag
- Awaiting user completion of Task 3 (environment variable setup) before hooks go live
- Once Task 3 is confirmed, Phase 01 is fully complete

---
*Phase: 01-foundation*
*Completed: 2026-04-02*
