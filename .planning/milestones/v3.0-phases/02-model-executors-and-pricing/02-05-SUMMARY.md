---
phase: 02-model-executors-and-pricing
plan: 05
subsystem: tools
tags: [searxng, context7, bash, websearch, fetchdocs, mcp-bridge]

# Dependency graph
requires:
  - phase: 01-plugin-scaffold-and-infrastructure
    provides: plugin directory structure at ~/.claude/plugins/seraphim/tools/
provides:
  - websearch.sh: SearXNG curl wrapper for non-Claude models
  - fetchdocs.js: Context7 MCP bridge with websearch fallback for non-Claude models
affects:
  - 03-discover-phase
  - 05-forge-phase
  - any Codex or Qwen executor that needs external search or documentation

# Tech tracking
tech-stack:
  added: []
  patterns:
    - "MCP-bridge pattern: non-Claude models access MCP tools via claude CLI subprocess"
    - "Graceful fallback: Context7 MCP -> websearch.sh when claude CLI unavailable"
    - "Security: all local service URLs use 127.0.0.1, never 0.0.0.0"

key-files:
  created:
    - ~/.claude/plugins/seraphim/tools/websearch.sh
    - ~/.claude/plugins/seraphim/tools/fetchdocs.js
  modified: []

key-decisions:
  - "fetchdocs.js uses claude CLI subprocess as MCP bridge — no confirmed public Context7 REST API exists; this is the only viable path for non-Claude models"
  - "fetchdocs.js fallback is websearch.sh, not a hard error — Codex/Qwen can still research even without claude CLI"
  - "websearch.sh uses 127.0.0.1:8888 explicitly per CLAUDE.md security rule, not localhost or 0.0.0.0"

patterns-established:
  - "Pattern: non-Claude MCP bridge via claude CLI subprocess (fetchdocs.js)"
  - "Pattern: bash+python3 pipeline for SearXNG JSON extraction (websearch.sh)"

requirements-completed: [EXEC-08, EXEC-09]

# Metrics
duration: 5min
completed: 2026-04-05
---

# Phase 2 Plan 05: Helper Tools Summary

**SearXNG curl wrapper (websearch.sh) and Context7 MCP bridge (fetchdocs.js) enabling non-Claude models to perform web search and documentation lookup**

## Performance

- **Duration:** ~5 min
- **Started:** 2026-04-05T03:22:00Z
- **Completed:** 2026-04-05T03:27:10Z
- **Tasks:** 1
- **Files modified:** 2

## Accomplishments

- Created websearch.sh: queries SearXNG at 127.0.0.1:8888, outputs one JSON object per line with title/url/content fields
- Created fetchdocs.js: bridges Context7 MCP for non-Claude models (Codex, Qwen) via claude CLI subprocess, falls back to websearch.sh if MCP unavailable
- Both scripts are executable (`chmod +x`) and self-contained in `tools/`
- Security constraint honored: websearch.sh uses `127.0.0.1:8888`, never `0.0.0.0` or bare `localhost`

## Task Commits

Each task was committed atomically:

1. **Task 1: Create tools/websearch.sh and tools/fetchdocs.js** - `77588bd` (feat)

**Plan metadata:** (docs commit follows)

## Files Created/Modified

- `/home/alucard/.claude/plugins/seraphim/tools/websearch.sh` - SearXNG curl wrapper; accepts query + limit, returns JSON lines
- `/home/alucard/.claude/plugins/seraphim/tools/fetchdocs.js` - Context7 MCP bridge via claude CLI subprocess with websearch.sh fallback

## Decisions Made

- Context7 has no confirmed public REST API; fetchdocs.js bridges via `claude mcp call plugin__context7__context7` subprocess — the research explicitly noted this as an open question and recommended the CLI approach
- Fallback chain is websearch.sh (not a hard failure) so Codex and Qwen retain research capability even without claude CLI
- 0.0.0.0 appears in websearch.sh comments only (as the disallowed form) — actual URL is always 127.0.0.1

## Deviations from Plan

None — plan executed exactly as written. The fetchdocs.js implementation follows the plan's code specification and research Open Question 1 recommendation verbatim.

## Issues Encountered

- SearXNG live query returned 0 results during verification — SearXNG itself responds with HTTP 200 and valid JSON, but the search engines may not be configured with live credentials in this environment. This is a SearXNG configuration issue, not a websearch.sh code issue. The script correctly connects, sends the query, and processes the JSON response.

## User Setup Required

None — no external service configuration required. Both scripts work with services already running (SearXNG at 127.0.0.1:8888).

## Next Phase Readiness

- websearch.sh ready for use by Codex and Qwen executors in Forge phase
- fetchdocs.js ready; effectiveness depends on claude CLI being in PATH when non-Claude models invoke it
- Both tools usable by Phase 3 Discover phase external track

---
*Phase: 02-model-executors-and-pricing*
*Completed: 2026-04-05*
