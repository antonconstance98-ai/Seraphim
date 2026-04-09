# Phase 2: Model Executors and Pricing - Context

**Gathered:** 2026-04-04
**Status:** Ready for planning

<domain>
## Phase Boundary

Fork codex-exec.js and minimax-exec.js from v2.0, build three new executors (gemini-exec.js, qwen-exec.js, perplexity-exec.js), implement pricing.js with per-provider cost functions, extend token-logger.js for all nine models, and create helper scripts (websearch.sh, fetchdocs.js).

</domain>

<decisions>
## Implementation Decisions

### Perplexity Executor
- **D-01:** Both MCP bridge and direct HTTP API, with fallback. Try MCP first (leverages existing Perplexity MCP config, zero additional setup). If MCP is unavailable, fall back to direct API via OpenAI SDK baseURL swap to `https://api.perplexity.ai` with `PERPLEXITY_API_KEY`.
- **D-02:** This dual-mechanism approach means perplexity-exec.js needs to detect MCP availability at runtime and route accordingly.

### Qwen Forge Mode
- **D-03:** Claude decides the structured output format (JSON actions vs markdown blocks) based on what Qwen handles most reliably. Design spec suggests JSON `{"action": "write", "path": "...", "content": "..."}` — Claude should validate whether Qwen 3.5 can produce this reliably.

### Executor Error Handling
- **D-04:** Executors return `{success: false, error_type, retriable: boolean}` — never throw. dispatch.js checks `retriable` flag and `error_type` to decide: retry same model, try profile-defined fallback, or surface to user.
- **D-05:** Error types: `unavailable` (model not running), `timeout` (exceeded limit), `rate_limit` (429/quota), `auth` (bad key), `parse` (malformed response), `unknown`.
- **D-06:** dispatch.js retry policy: retry on `rate_limit` (with exponential backoff), attempt fallback on `unavailable`/`timeout`, surface to user on `auth`/`unknown`.

### Claude's Discretion
- Qwen structured output schema specifics
- Gemini SDK integration patterns (search grounding, thinking mode)
- Helper script implementation details
- Fork adaptation specifics (path resolution, interface adaptation)

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Design Spec
- `docs/specs/2026-04-04-seraphim-v3-design.md` — Executor interface (`execute`, `stream`, `available`), model roster with access methods, Qwen wrapper spec, Gemini API spec

### Research
- `.planning/research/STACK.md` — `@google/genai@1.48.0` (NOT deprecated `@google/generative-ai`), Qwen model tag `qwen3:30b-a3b-q4_K_M`, Perplexity base URL has no `/v1` suffix, ollama HTTP API at localhost:11434
- `.planning/research/PITFALLS.md` — Qwen cold-start 13-46s (needs 180s timeout + warm-up probe), Gemini Flash rate limits (10 RPM on paid tier), nine models use four incompatible token schemas
- `.planning/research/ARCHITECTURE.md` — Executor fork changes (remove runWithFallback from minimax, remove fallback chain from codex), path audit requirement

### Existing Code (Fork Sources)
- `~/.claude/hooks/codex-exec.js` — Codex CLI wrapper (fork and adapt)
- `~/.claude/hooks/minimax-exec.js` — MiniMax API wrapper (fork and adapt)
- `~/.claude/hooks/codex-pricing.js` — Pricing module (fork and extend for 9 models)
- `~/.claude/hooks/codex-token-logger.js` — Token logger (fork and extend)

### Phase 1 Context
- `.planning/phases/01-plugin-scaffold-and-infrastructure/01-CONTEXT.md` — dispatch.js resolution order, custom profile support

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `codex-exec.js` — `runCodexExec()` function with 300s timeout, SIGTERM/SIGKILL, JSONL parsing
- `minimax-exec.js` — `runMinimax()` with OpenAI SDK baseURL swap, `callWithRetry()` with exponential backoff, AbortController timeout
- `codex-pricing.js` — `CODEX_PRICING` object, `computeCost()`, `computeCodexCostStrict()`, MiniMax pricing constants

### Established Patterns
- OpenAI SDK v6.33.0 baseURL swap for MiniMax (reuse for Perplexity)
- `callWithRetry()` with configurable retries and exponential backoff
- `available()` pattern not yet standardized — needs to be defined per executor

### Integration Points
- Executors loaded by dispatch.js via lazy `require()`
- Each executor call logged by token-logger.js
- pricing.js consumed by token-logger and cost-reporter
- Helper scripts called by Codex/Qwen/Gemini during Discover phase

</code_context>

<specifics>
## Specific Ideas

No specific requirements — open to standard approaches for executor implementation.

</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope

</deferred>

---

*Phase: 02-model-executors-and-pricing*
*Context gathered: 2026-04-04*
