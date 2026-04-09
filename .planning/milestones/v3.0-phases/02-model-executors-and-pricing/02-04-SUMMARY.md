---
phase: 02-model-executors-and-pricing
plan: 04
subsystem: executors
tags: [qwen, perplexity, ollama, local-inference, mcp, web-search, executor]
dependency_graph:
  requires: ["02-01"]
  provides: ["qwen-exec.js", "perplexity-exec.js"]
  affects: ["dispatch.js", "Balanced/Budget profiles", "Discover phase"]
tech_stack:
  added: []
  patterns:
    - "ollama OpenAI-compat endpoint at localhost:11434/v1 for local Qwen inference"
    - "inference probe (max_tokens:1) for available() — loads model into VRAM, catches GPU failures"
    - "Forge mode structured JSON output via FORGE_SYSTEM_PROMPT"
    - "Perplexity baseURL without /v1 suffix (adding /v1 causes 404)"
    - "MCP-first pattern: return mcpRequest object when opts.mcpAvailable=true; caller invokes MCP tool"
key_files:
  created:
    - ~/.claude/plugins/seraphim/executors/qwen-exec.js
    - ~/.claude/plugins/seraphim/executors/perplexity-exec.js
  modified: []
decisions:
  - "available() uses inference probe not /api/tags — forces VRAM load, catches cold-start failures that tag listing misses"
  - "Perplexity baseURL is https://api.perplexity.ai with no /v1 suffix — Perplexity appends /chat/completions directly off the base"
  - "MCP path returns mcpRequest object to caller — MCP tools inaccessible from standalone Node.js (Pitfall 7)"
metrics:
  duration: "108s"
  completed: "2026-04-05"
  tasks_completed: 2
  tasks_total: 2
  files_created: 2
  files_modified: 0
---

# Phase 02 Plan 04: Qwen + Perplexity Executors Summary

**One-liner:** Qwen local executor with inference probe and Forge mode + Perplexity dual-path executor using MCP-first pattern with direct API fallback and citation extraction.

## Tasks Completed

| Task | Name | Commit | Files |
|------|------|--------|-------|
| 1 | Create qwen-exec.js with inference probe and Forge mode | e2b5a05 | executors/qwen-exec.js |
| 2 | Create perplexity-exec.js with MCP+API dual path | 4e45fa1 | executors/perplexity-exec.js |

## What Was Built

### qwen-exec.js

Qwen 3.5-27B local executor via the ollama OpenAI-compatible endpoint at `localhost:11434/v1`. Key implementation details:

- `MODEL_TAG = 'qwen3:30b-a3b-q4_K_M'` — correct ollama tag, not the models.json alias
- `EXECUTE_TIMEOUT_MS = 180000` (180s) with `PROBE_TIMEOUT_MS = 60000` (60s)
- `NUM_CTX = 32768` passed as extra field; ollama accepts it directly
- **Inference probe in available()**: sends `max_tokens:1` to force model load into VRAM. `/api/tags` would return the model name even when the GPU is absent or VRAM is insufficient — this probe catches that failure class
- **Forge mode**: prepends `FORGE_SYSTEM_PROMPT` requesting JSON-only output, then `parseForgeActions()` splits on newlines and JSON.parses each `{`-prefixed line
- **Error categorisation**: ECONNREFUSED → `unavailable/non-retriable`, timeout/abort → `timeout/retriable`, other → `unknown/non-retriable`
- Cost: always `computeFreeCost()` = 0 (local GPU, zero marginal cost)

### perplexity-exec.js

Perplexity Sonar executor implementing the MCP-first + direct API fallback pattern (D-01/D-02):

- **MCP path**: when `opts.mcpAvailable === true`, returns a `mcpRequest` object `{ tool, args }` for the Claude command caller to invoke. The executor cannot call MCP directly from Node.js
- **Direct API path**: OpenAI SDK with `baseURL: 'https://api.perplexity.ai'` — no `/v1` suffix (adding it causes 404)
- **Citation extraction**: reads `response.citations || []` (Perplexity extension field), appends as `## Sources` section with numbered references
- **available()**: returns `!!process.env.PERPLEXITY_API_KEY` — MCP availability is checked at call time via opts, not here
- **Error categorisation**: 429 → `rate_limit/retriable`, 401/403 → `auth/non-retriable`, timeout → `timeout/retriable`

## Verification Results

All plan-level checks passed:

```
Both loaded OK
qwen available: false   (correct — ollama not running on this machine)
No old hook path references (good)
```

The `available()` returning `false` for Qwen is the correct behaviour — GPU (RTX 3090) is in transit. The executor exists and fails gracefully exactly as required by the plan.

## Deviations from Plan

None — plan executed exactly as written.

## Known Stubs

None — both executors are fully wired. Qwen will return `available: false` until the RTX 3090 is installed and ollama loads the model. This is expected behaviour, not a stub.

## Self-Check: PASSED
