# Pitfalls Research

**Domain:** Six-phase multi-model plugin pipeline added to an existing three-model hook system (Seraphim v3.0)
**Researched:** 2026-04-04
**Confidence:** HIGH for plugin manifest and hook migration pitfalls (verified against official Claude Code docs); HIGH for ollama/GPU pitfalls (verified against InsiderLLM, ollama GitHub issues); MEDIUM for Gemini rate limit specifics (Google docs confirmed shape but numeric limits require AI Studio console check); HIGH for feedback loop control (verified against Anthropic agent guidance, LoopJar production reports); HIGH for token logging normalization (verified against multiple multi-provider logging sources)

---

## Critical Pitfalls

---

### Pitfall 1: Hook Double-Registration — Plugin hooks.json Conflicts With Explicit Manifest Declaration

**What goes wrong:**
The plugin spec places hook config at `hooks/hooks.json`. Claude Code auto-discovers this file by convention for any installed plugin. If `plugin.json` also declares `"hooks": "./hooks/hooks.json"` (or any path pointing to the same file), Claude Code detects the duplicate and throws a `conflicting manifests` error, silently dropping all plugin hooks. The plugin loads, commands work, but hooks never fire — no error at runtime, only at validation.

**Why it happens:**
Developers coming from the v1.0/v2.0 hook system assume hooks must be explicitly registered. In the plugin system, the standard location is auto-loaded; explicit declaration is only needed for non-standard paths or additional hook files.

**How to avoid:**
- Do NOT declare `"hooks": "./hooks/hooks.json"` in `plugin.json` if that is the standard location.
- Only use the `hooks` field in `plugin.json` to reference additional hook files beyond `hooks/hooks.json`.
- Run `claude plugin validate` after every change to `plugin.json` before testing.
- Use `claude --debug` on first install — look for "loading plugin" output confirming hooks registration.

**Warning signs:**
- `Bug: Duplicate hooks file error` in Claude Code output.
- Hook scripts never execute despite being executable and syntactically correct.
- `claude plugin validate` passes but no hooks appear under `/plugin` Errors tab.

**Phase to address:** Plugin scaffold phase (whichever phase creates plugin.json and hooks/hooks.json). The validate step must be part of the phase exit criteria.

---

### Pitfall 2: Hooks From Existing System Still Fire — Redundant Hook Interference

**What goes wrong:**
Seven hooks are planned for consolidation into the Seraphim pipeline: `codex-review-gate.js`, `codex-plan-reviewer.js`, `codex-multi-round-reviewer.js`, `minimax-post-scan.js`, `minimax-compress.js`, `codex-router.js`, `codex-wave-validator.js`. If these remain active in `~/.claude/settings.json` during v3.0 development, they run alongside the new plugin hooks. The result: every Forge task triggers both the old PostToolUse scanner AND the new Crucible adversarial pass. Cost doubles. Timing conflicts cause race conditions on shared JSONL log files. The old hooks reference paths that may not exist post-migration.

**Why it happens:**
Hook cleanup is treated as the last step ("remove old hooks after new system is stable"). In practice, the migration window extends indefinitely because the old hooks are providing a safety net. The hooks are never removed.

**How to avoid:**
- Disable the seven consolidation-target hooks in `~/.claude/settings.json` at the START of the phase that implements their pipeline replacements, not at the end.
- Keep a backup: copy the old hook entries to `~/.claude/hooks/archive/` before removing from settings.
- Keep `token-logger.js` and `session-start.js` active until their plugin forks are validated, then swap atomically.
- Test with `SERAPHIM_HOOKS_ONLY=1` env var that the old hooks check for and skip — allows gradual cutover.

**Warning signs:**
- Token logs show double-counted costs for the same tool call.
- Forge phase takes 2x expected time with no explanation.
- JSONL log files have write conflicts (partial lines, JSON parse errors at read time).

**Phase to address:** Hook consolidation phase — this must be a discrete phase, not an afterthought. Scope it explicitly: "disable old hooks, enable plugin equivalents, validate no duplicates."

---

### Pitfall 3: ollama Cold-Start Timeout — Qwen Misreported as Unavailable

**What goes wrong:**
When ollama has not been used recently, the Qwen 3.5-27B model must be loaded from disk into VRAM. On an RTX 3090 with a 27B Q4 model (~17GB), this load takes 13-46 seconds. If `qwen-exec.js` uses the same 30-second default timeout as cloud executors, the first request times out mid-load. The `available()` check passes (ollama process is running, `/api/tags` responds), but `execute()` times out. The dispatch layer interprets this as Qwen being unavailable and falls back to the next model in the profile — which may be a paid cloud API. The user gets a cloud bill for what should have been free local inference, with no indication anything unusual happened.

**Why it happens:**
The `available()` check only verifies ollama is running and the model is pulled. It does not distinguish between "model is warm" and "model needs to be loaded." The 120s timeout specified in the design spec is correct — but if the executor implementation inherits a shorter default from a shared HTTP client config, the cold-start hits the shorter value.

**How to avoid:**
- In `qwen-exec.js`, set HTTP timeout to 180s (larger than the 120s design spec value to give margin).
- Implement a separate warm-up pre-request: before the first real Qwen call per session, send a minimal prompt (`"ping"`) and wait up to 60s. Only proceed with real requests once this succeeds.
- Log cold-start duration when it exceeds 10s so the user sees "Qwen warming up (23s)..." rather than silence.
- The `available()` method should ping `/api/generate` with a timeout of 60s, not just check `/api/tags`.
- Do not share an HTTP client instance between Qwen executor and cloud executors — Qwen needs independent timeout config.

**Warning signs:**
- First Qwen call per session silently routes to a cloud fallback.
- Balanced/Budget profile costs are higher than expected without operator-visible explanation.
- ollama process shows GPU utilization spike immediately after a "timeout" is logged.

**Phase to address:** Qwen executor implementation phase. The warm-up pre-request and 180s timeout must be in the acceptance criteria for `qwen-exec.js`, not discovered during integration testing.

---

### Pitfall 4: VRAM Exhaustion Degrades Inference Mid-Session — Context Kills the Budget

**What goes wrong:**
Qwen 3.5-27B at Q4 requires ~17GB VRAM at 4K context. The design spec caps context at 32K, which pushes VRAM to approximately 19GB — leaving only 5GB headroom on the RTX 3090's 24GB. However, during a real Forge session, blueprint.md is loaded into context, plus prior task outputs, plus the current task. Context accumulates across the full session. At ~64K tokens, VRAM requirement jumps to ~21GB. At 96K+ tokens, ollama starts offloading KV cache layers to CPU RAM via hybrid mode, crossing PCIe twice per token — inference speed drops from 34 tok/s to under 5 tok/s. The 120s executor timeout now fires on legitimate inference rather than cold start.

**Why it happens:**
The design spec sets a 32K token context cap in `qwen-exec.js`. However, the cap applies per-call, not across the session. If the caller constructs a prompt that includes the full blueprint (which can be long) plus multi-task forge logs, the per-call context can exceed 32K even with truncation logic. Additionally, ollama's KV cache is cumulative across calls in the same model session unless explicitly cleared.

**How to avoid:**
- Enforce context budget at the dispatcher level, not just in the executor: count tokens in the full prompt before sending. If over 28K tokens (leaving 4K buffer below the 32K cap), truncate the oldest phase context first.
- Always pass `num_ctx: 32768` explicitly in the ollama API call to prevent ollama from using its default (which may be higher).
- Between Forge tasks, send `{"keep_alive": 0}` to unload the model and reload fresh, clearing the KV cache. This adds cold-start latency between tasks but prevents VRAM accumulation.
- Monitor VRAM usage during Forge sessions: log `nvidia-smi --query-gpu=memory.used --format=csv,noheader` at each checkpoint and alert if over 20GB.

**Warning signs:**
- Forge inference slows down progressively over a long session.
- Token-per-second rate drops below 10 tok/s mid-session.
- ollama logs show `offloading to CPU` or layer split messages.
- GPU memory climbs above 20GB during a Balanced/Budget profile session.

**Phase to address:** Qwen executor and Forge checkpoint phase. KV cache management must be in the spec, not discovered empirically.

---

### Pitfall 5: Gemini Rate Limit 429s With No Retry Logic — Judge Phase Silently Fails

**What goes wrong:**
Gemini 3 Flash is the Judge model in Performance and Moderate profiles. The Gemini API enforces RPM and TPD limits that were reduced 50-80% in December 2025. A 429 rate-limit error from Gemini is a retriable error — back off and retry. However, if `gemini-exec.js` treats HTTP 429 the same as HTTP 500 (a hard failure), the Judge phase exits immediately, `judgment.md` is never written, and the pipeline stalls at Judge with an executor error. The dispatch fallback chain activates (if configured for Judge), but falls back to a model that may not have adversarial thinking mode, degrading quality without notification.

**Why it happens:**
429 and 500 are both HTTP errors. Without explicit retry handling for 429, the error propagates up. Gemini's rate limits apply per-project (not per key), so if a developer tests repeatedly within a short window, they hit the limit quickly. Free tier Gemini Flash allows only 10 RPM and 250 RPD — easy to exhaust during iterative development.

**How to avoid:**
- In `gemini-exec.js`, implement exponential backoff with jitter specifically for HTTP 429: start at 5s, double up to 60s, max 3 retries.
- Distinguish 429 (rate limit, retriable) from 500/503 (server error, retriable) from 400 (client error, not retriable) from 401 (auth error, not retriable — surface to user immediately).
- Log all 429s with the retry count and wait duration so the operator can see "Gemini rate-limited, retry 2/3 (30s wait)".
- During development, use a separate test Google Cloud project to avoid burning the same quota used by production sessions.
- Track daily request count per model in the token log; alert at 80% of daily limit before hitting it.

**Warning signs:**
- Judge phase fails within seconds with no model output.
- Token log shows 0 tokens used for Gemini calls that should have generated output.
- A pattern of Judge failures around the same time each day (daily quota reset at midnight Pacific).

**Phase to address:** Gemini executor implementation phase. Retry logic is not optional — it is part of the executor spec. The phase acceptance criteria must include a 429-injection test.

---

### Pitfall 6: Feedback Loop Counter Not Persisted — Loop Cap Fails Across Sessions

**What goes wrong:**
The design spec allows max 2 loops for Judge->Envision and max 2 loops for Crucible->Forge. The loop counter is tracked in memory during a pipeline run. If the user interrupts the session after loop 1 (Ctrl-C, crash, timeout), the counter resets. On the next session, the loop restarts at 0. If the underlying quality problem is not resolved, the system loops again indefinitely across sessions — technically respecting the "max 2" per-session limit while violating the intent of the global cap. In an adversarial scenario: Crucible consistently fails, user restarts pipeline, Forge runs again, Crucible fails again, Forge runs again — potentially triggering expensive Forge+Crucible cycles repeatedly without escalating to the human.

**Why it happens:**
In-memory loop counters are the path of least resistance. Persisting them to `phase-state.js` / `.seraphim/phases/{N}/` requires explicit design. Teams defer persistence as "we'll add it later" and it never gets added.

**How to avoid:**
- Store loop counts in `.seraphim/phases/{N}/state.json` at every loop increment, not in memory.
- `phase-state.js` must expose `incrementLoop(phase, loopType)` and `getLoopCount(phase, loopType)` with persistent backing.
- Before starting any phase that can loop (Judge, Forge checkpoint, Crucible), read the persisted loop count. If already at max, surface to human before executing.
- The state file must include a `reset_at` timestamp so old loop counts from previous milestone runs do not carry over to new ones.

**Warning signs:**
- Crucible sends back to Forge for the "first time" but the forge-log.md already shows two prior Forge passes.
- Cost per pipeline run increases with each session restart on the same phase number.
- The human is never asked to intervene despite repeated Crucible failures.

**Phase to address:** Phase state infrastructure phase. This must be built before any feedback loop is implemented, not alongside it.

---

### Pitfall 7: Dispatch Routing Failure Is Silent — Wrong Model Executes Without Notification

**What goes wrong:**
`dispatch.js` resolves the config, applies overrides, and calls the target executor. If the executor's `available()` returns false, dispatch falls back down the profile's fallback chain. This is the correct behavior. However: if `available()` returns true but `execute()` fails partway through (network drop, model error, malformed response), dispatch may log the failure and silently retry with the fallback without informing the user. The human sees a phase output — it just came from a different model than they configured. Quality, cost, and auditing assumptions are violated.

**Why it happens:**
Fallback chains are designed to be transparent to the user ("it just works"). The implicit assumption is that the fallback model produces equivalent quality. For Seraphim's phase assignments, this is false: the Judge model assignment (Gemini Flash) was chosen specifically for its 90.4% GPQA adversarial reasoning. Falling back to MiniMax for Judge without notification changes the quality contract.

**How to avoid:**
- Every fallback activation MUST write a visible log entry: `[SERAPHIM] DISPATCH FALLBACK: judge phase — gemini-3-flash failed (timeout), falling back to minimax-m2.7. Quality impact: reduced adversarial reasoning capability.`
- The token log entry for every execution must record both the `requested_model` and `actual_model` fields. These should differ only when a fallback occurred.
- `decisions.jsonl` must record fallback events with reason code (timeout, auth, rate-limit, unavailable) for adaptive intelligence analysis.
- Do not silently swallow errors in executor `execute()` — always return `{ success: false, error: { code, message, retriable } }` so dispatch can make informed decisions.
- The phase output file (e.g., `judgment.md`) must include a header noting which model produced it: `<!-- model: minimax-m2.7 (fallback from gemini-3-flash: timeout) -->`.

**Warning signs:**
- Adaptive intelligence recommendations cite models that the operator did not knowingly use.
- `requested_model` and `actual_model` fields are identical in all token log entries (fallback events are not being recorded).
- Phase quality varies unexpectedly without any config change.

**Phase to address:** Dispatch implementation phase. The `requested_model`/`actual_model` split and fallback logging must be in the dispatch spec, not added later.

---

### Pitfall 8: Token Logging Breaks Across Nine Models — Different Response Schemas, Negative Costs

**What goes wrong:**
Nine models use four different response schemas for token counts. Anthropic subagents (Opus, Sonnet, Haiku) use `usage.input_tokens` and `usage.output_tokens`. Gemini uses `usageMetadata.promptTokenCount` and `usageMetadata.candidatesTokenCount`. MiniMax uses `usage.prompt_tokens` and `usage.completion_tokens` (OpenAI-compatible but with different cached-token semantics). Qwen via ollama uses `prompt_eval_count` and `eval_count`. Codex CLI outputs JSON with `usage.input_tokens`/`usage.output_tokens` but with Codex-specific cached-token fields. If `pricing.js` normalizes costs using a shared formula, Anthropic's cached-token credits (`usage.cache_read_input_tokens`) produce a negative cost delta when subtracted without provider-specific handling. This was flagged as HIGH severity in the Phase 10 adversarial review for the v2.0 system and will reoccur at larger scale in v3.0 with nine models.

**Why it happens:**
The unified executor interface returns `{ tokens, cost }` — but if each executor computes its own `cost` using the same helper function, the helper must know which provider it's computing for. Passing a `provider` string is easy to forget, and the default formula silently produces wrong numbers rather than erroring.

**How to avoid:**
- `pricing.js` must expose per-provider cost functions: `computeCost(provider, inputTokens, outputTokens, cacheReadTokens, cacheWriteTokens)`. Never a generic `computeCost(inputTokens, outputTokens)`.
- Every executor is responsible for extracting its provider's specific fields from the raw API response and passing them to the correct pricing function. No sharing of extraction logic across providers.
- The token log schema must include `raw_usage` (the original API response fields) alongside the normalized `tokens_in`, `tokens_out`, `cache_read`, `cost`. This enables auditing without re-fetching.
- On startup, `pricing.js` should emit a validation log: all nine models listed with their current pricing and the formula used. This makes pricing drift visible.
- Monthly reconciliation: compare `sum(token_log.cost)` against actual API invoices for each provider. Any discrepancy over 5% requires investigation before the next cycle.

**Warning signs:**
- Cost entries in token log are negative for Anthropic calls.
- MiniMax cost per session diverges from the MiniMax invoice by more than 10%.
- The `cost` field is zero for Qwen calls (ollama response fields not mapped).
- All nine models show identical cost-per-token ratios (shared formula not differentiated).

**Phase to address:** Pricing and token logging infrastructure phase — must be built and validated before any executor that uses it. Each executor's tests must include a cost-calculation assertion against the known pricing table.

---

### Pitfall 9: Plugin manifest.json Placed in Wrong Directory — Components Not Discovered

**What goes wrong:**
The official plugin structure requires `plugin.json` to be at `.claude-plugin/plugin.json` (inside a `.claude-plugin/` subdirectory at the plugin root). All other directories — `commands/`, `agents/`, `hooks/`, `executors/`, `lib/` — must be at the plugin root itself, NOT inside `.claude-plugin/`. If the developer places `plugin.json` directly at the plugin root (or places `commands/` inside `.claude-plugin/`), Claude Code either fails to load the manifest or silently ignores components. This is the most common plugin structure mistake documented in the Claude Code issue tracker.

**Why it happens:**
The design spec shows the plugin structure without the `.claude-plugin/` subdirectory. Developers read the spec and create `~/.claude/plugins/seraphim/plugin.json` instead of `~/.claude/plugins/seraphim/.claude-plugin/plugin.json`. This is a silent failure: Claude Code loads the plugin by directory name but no manifest is found, so it uses defaults. Commands in `commands/` may still be discovered, but metadata, custom paths, and hook declarations in the manifest are ignored.

**How to avoid:**
- The plugin scaffold (first commit) must create the `.claude-plugin/` directory and `plugin.json` inside it.
- Immediately after creating the scaffold, run `claude --plugin-dir ~/.claude/plugins/seraphim --debug` and confirm the manifest is loaded ("loading plugin seraphim from manifest").
- Add a CI/pre-commit check: `test -f ~/.claude/plugins/seraphim/.claude-plugin/plugin.json` as a sanity guard.
- The design spec directory tree is misleading — update it to show `.claude-plugin/plugin.json` explicitly.

**Warning signs:**
- `/seraphim:discover` command exists (auto-discovered from `commands/`) but `/seraphim:set-profile` does not (requires manifest to register non-standard components).
- `claude plugin validate` reports "no manifest found" but `claude --debug` shows the plugin loading.
- Plugin hooks never fire despite hooks.json being syntactically correct and executable.

**Phase to address:** Plugin scaffold phase (day one). Getting the directory structure right is a prerequisite for every subsequent phase.

---

### Pitfall 10: Feedback Loop Condition Evaluated by the Model Being Looped — Biased Termination

**What goes wrong:**
The Judge->Envision loop fires when "zero approaches survive" in `judgment.md`. This condition is evaluated by parsing the output of the Judge model. If the parsing logic is lenient (accepts "CONDITIONAL" as a survivor), the loop terminates. If it is strict (only "SURVIVES" triggers termination), the loop may fire unnecessarily. More critically: if the Crucible->Forge loop trigger is determined by Crucible's own output (`crucible.md` says FAIL), the adversarial model can oscillate — finding new issues each pass that were introduced by the previous Forge fix, creating a legitimate infinite loop within the hard cap. Two full Forge+Crucible passes can be expensive ($3-8 each in Performance profile, 2 × max = $16 possible before the human is asked).

**Why it happens:**
The loop termination condition is qualitative (PASS/FAIL, SURVIVES/FATAL FLAW) and depends on model output interpretation. The parsing of structured output from LLMs is fragile. A model under adversarial pressure may introduce hedging language ("the implementation partially meets...") that breaks the parser in either direction.

**How to avoid:**
- Define the loop termination schema precisely: `judgment.md` must use machine-readable markers, not prose. Example: `<!-- STATUS: SURVIVES|FATAL_FLAW|CONDITIONAL -->` per approach, followed by prose explanation. Parse the comment, not the prose.
- Before triggering a loop-back, estimate the cost of the next iteration and surface it to the user: "Crucible found issues. Running Forge again will cost approximately $4-6. Proceed? [y/N]".
- Hard limit: before spending over $10 on loop iterations for a single pipeline run, pause and require explicit human approval.
- Separate issue tracking from loop triggering: Crucible should log specific issues to `crucible.md`; a separate coordinator function decides whether those issues warrant a Forge loop or can be addressed inline. The adversarial model should not control its own loop termination.

**Warning signs:**
- Loop count reaches max without the underlying issue being resolved.
- Cost spikes on sessions where Crucible is active in Performance profile.
- `judgment.md` contains "CONDITIONAL" on all approaches but the pipeline does not loop (parser bug in termination condition).

**Phase to address:** Phase command implementation (Judge, Crucible). The machine-readable output schema and cost-gate must be designed before the feedback loop logic, not after.

---

### Pitfall 11: Migrating from Hook Path to Plugin Path — Hardcoded Paths Break Forked Scripts

**What goes wrong:**
`codex-exec.js` and `minimax-exec.js` are forked from `~/.claude/hooks/`. The original hook scripts hardcode paths like `~/.claude/hooks/codex-pricing.js` or reference sibling files via relative paths (`../codex-token-logger.js`). After forking into `~/.claude/plugins/seraphim/executors/`, these relative references resolve to the old hook directory, silently loading the pre-fork version of the file. Pricing tables may be stale. Token logging goes to the old hook log, not the plugin's `token-log.jsonl`. The developer tests the plugin, sees it "works," but costs are logged in two places and the plugin dashboard shows zero data.

**Why it happens:**
Fork = copy the file, change the path. But `require()` calls with relative paths are easy to miss. Node.js resolves them from the file's location, not the caller's location — so a relative path in the forked file still works, just resolves to the wrong directory.

**How to avoid:**
- In every forked executor, replace ALL relative `require()` paths with `path.join(__dirname, '../lib/pricing.js')` style absolute-relative paths anchored to `__dirname`.
- Immediately after forking, run a path audit: `grep -n "require(" executors/*.js | grep "\.\."` and manually verify each result resolves to the intended plugin-internal file.
- The `CLAUDE_PLUGIN_ROOT` environment variable is available in hook processes. Use it in shell scripts as the anchor for all paths.
- Add an integration test that the forked executor loads with zero references to `~/.claude/hooks/`.

**Warning signs:**
- Token log at `~/.claude/hooks/` continues to grow during Seraphim sessions.
- Plugin dashboard shows zero cost data despite active sessions.
- Pricing table shows v2.0 model prices after v3.0 pricing updates (loaded stale fork).

**Phase to address:** Executor fork phase. The path audit must be an explicit exit criterion before integration tests begin.

---

## Technical Debt Patterns

Shortcuts that seem reasonable but create long-term problems.

| Shortcut | Immediate Benefit | Long-term Cost | When Acceptable |
|----------|-------------------|----------------|-----------------|
| In-memory loop counter | No file I/O overhead | Loop cap fails across interrupted sessions | Never — loop state must be persisted |
| Shared cost formula across all nine providers | Less code | Negative costs for Anthropic, zero costs for Qwen (wrong field names) | Never — per-provider functions required |
| Re-use hook timeout (30s) for Qwen executor | Consistent config | Cold-start silently routes to paid cloud fallback | Never — Qwen needs 180s minimum |
| Declare hooks in plugin.json AND keep hooks/hooks.json | Belt and suspenders | Duplicate detection error — hooks never fire | Never — pick one location |
| Evaluate feedback loop termination from prose output | Natural language output | Parser fragility causes false positives/negatives on loop trigger | Never — require machine-readable markers |
| Leave old redundant hooks active during migration | Safety net during transition | Double-triggers, JSONL write conflicts, doubled costs | Maximum 48 hours — disable old hooks immediately once plugin equivalent is validated |
| Probe ollama availability with /api/tags only | Fast startup check | Cold-start appears available but execute() times out | Never — probe with actual inference request |
| Store per-project state in plugin directory | Simpler paths | Plugin cache directory is wiped on update, losing phase state | Never — per-project state must be in `<project>/.seraphim/` |

---

## Integration Gotchas

Common mistakes when connecting to external services.

| Integration | Common Mistake | Correct Approach |
|-------------|----------------|------------------|
| Gemini API | Treat HTTP 429 as hard failure | Implement exponential backoff (5s → 10s → 30s → 60s, max 3 retries) with jitter; log each retry attempt |
| Gemini API | Use free-tier quota for iterative development | Create a separate Google Cloud project for development; track RPD consumption separately from production quota |
| ollama / Qwen | Set same timeout as cloud executors (30s) | Set HTTP timeout to 180s in qwen-exec.js; implement warm-up probe on first session use |
| ollama / Qwen | Assume context cap is enforced by caller | Always pass `num_ctx: 32768` explicitly in every ollama API call; enforce token count before sending |
| MiniMax | Reuse OpenAI SDK cost formula (`promptTokens - cachedTokens`) | MiniMax cached token accounting differs; use MiniMax-specific pricing function with `temperature: 0.01` (API rejects exactly 0) |
| Codex CLI | Parse stdout as JSON directly | Codex CLI JSON output may include progress lines before the final JSON object; parse the last complete JSON object only |
| Perplexity via MCP | Call MCP tool directly from executor | MCP tools are only accessible to Claude session, not to Node.js executor scripts; use perplexity-exec.js as a bridge that formats a Claude tool-use request and returns parsed results |
| Plugin hooks | Write hook scripts without executable bit | Every hook script must be `chmod +x`; hooks silently fail if not executable |
| Plugin path variables | Use hardcoded absolute paths | Use `${CLAUDE_PLUGIN_ROOT}` in hooks.json and `.mcp.json`; use `__dirname` in Node.js scripts |

---

## Performance Traps

Patterns that work at small scale but fail as usage grows.

| Trap | Symptoms | Prevention | When It Breaks |
|------|----------|------------|----------------|
| Qwen context accumulation across Forge tasks | Inference slows from 34 tok/s to <5 tok/s mid-session | Send keep_alive=0 between Forge tasks to reset KV cache; enforce 28K token hard cap per call | When blueprint + 3+ completed task outputs exceed 40K tokens |
| Unbounded decisions.jsonl growth | Adaptive intelligence analysis time grows linearly | Implement 90-day rolling window; archive older entries to `decisions-archive.jsonl` | After ~10,000 decisions (~6 months of active use) |
| Synchronous checkpoint.js blocking Forge pipeline | Forge appears to hang between tasks | checkpoint.js must run tests with a 30s timeout and fail-fast; never block indefinitely on test suite | First time a test suite has a hanging test |
| Loading full blueprint into every executor call | Each Forge task call hits 32K token cap immediately | Pass only the current task excerpt plus relevant context from blueprint; use a context window budget function | When blueprint.md exceeds 5,000 tokens (~medium-complexity project) |
| Session-start hook re-loading full decisions.jsonl | Session start takes 10+ seconds | Cache the analysis result; only re-analyze if decisions.jsonl has grown since last analysis | When decisions.jsonl exceeds ~500 entries |

---

## Security Mistakes

Domain-specific security issues beyond general security hygiene.

| Mistake | Risk | Prevention |
|---------|------|------------|
| Logging full prompt content to decisions.jsonl | Sensitive code, credentials, PII in adaptive intelligence training data | Log only derived features: phase, model, token_count, latency_ms, outcome, loop_count. Never log prompt text. |
| Sending project code through MiniMax API | MiniMax privacy policy does not guarantee data isolation; design spec already notes "never send credentials/PII to MiniMax" | Before any MiniMax call, strip file paths, credentials, and PII from the prompt. Send only the structural/logical content needed for the adversarial review. |
| Storing GEMINI_API_KEY in .seraphim/config.json | Key committed to version control if config.json is not in .gitignore | API keys in environment variables only; config.json contains profile name and overrides, never credentials |
| Plugin writing state to CLAUDE_PLUGIN_ROOT | Plugin cache directory is wiped on plugin update | All mutable state goes to `<project>/.seraphim/` or `${CLAUDE_PLUGIN_DATA}`; CLAUDE_PLUGIN_ROOT is read-only runtime |
| qwen-exec.js exposing ollama API on non-localhost | ollama default binds to 127.0.0.1; if changed, model accessible on LAN | Always verify ollama bind address is 127.0.0.1 in qwen-exec.js startup check; reject non-localhost base URLs |

---

## "Looks Done But Isn't" Checklist

Things that appear complete but are missing critical pieces.

- [ ] **Plugin manifest:** Verify `plugin.json` is at `.claude-plugin/plugin.json`, not at the plugin root. Run `claude plugin validate` and confirm no errors.
- [ ] **Hook deduplication:** Verify old hooks (`codex-review-gate.js`, `minimax-post-scan.js`, etc.) are removed from `~/.claude/settings.json`. Run a test session and confirm no double-logging in token-log.jsonl.
- [ ] **Qwen cold-start:** Verify first Qwen call per session uses the warm-up probe and completes within 60s without falling back to a cloud model. Check token log `actual_model` field on first Balanced profile session.
- [ ] **Fallback logging:** Verify that when a model is made artificially unavailable, the dispatch fallback is logged with both `requested_model` and `actual_model`. The phase output file header must reflect the actual model used.
- [ ] **Loop counter persistence:** Verify that killing a session mid-loop and restarting does not reset the loop counter. Check `.seraphim/phases/{N}/state.json` exists after a loop iteration.
- [ ] **Token cost accuracy:** Verify Anthropic cache-read token cost produces a positive or zero cost adjustment (not negative). Verify Qwen ollama calls show non-zero token counts. Verify MiniMax costs match the $0.30/$1.20 per Mtok pricing table.
- [ ] **Context cap enforcement:** Verify that when a prompt exceeds 28K tokens, the executor truncates and logs a warning rather than sending the oversized request to ollama.
- [ ] **Gemini retry:** Verify that a 429 response from Gemini triggers exponential backoff and is retried, not immediately failed. Inject a synthetic 429 in the test suite.
- [ ] **Path isolation:** Verify no executor script loads files from `~/.claude/hooks/`. Run `node -e "require('./executors/codex-exec.js')"` from the plugin root and confirm no ENOENT errors for hook-path references.

---

## Recovery Strategies

When pitfalls occur despite prevention, how to recover.

| Pitfall | Recovery Cost | Recovery Steps |
|---------|---------------|----------------|
| Hook double-registration (both plugin and settings.json fire) | LOW | 1. Remove hook declarations from `plugin.json` — keep only hooks/hooks.json. 2. Run `claude plugin validate`. 3. Restart session and confirm single execution per event. |
| Old hooks still firing alongside new pipeline | LOW | 1. Comment out seven redundant hook entries in `~/.claude/settings.json`. 2. Restart session. 3. Confirm token log shows single-source entries. 4. Archive old hook scripts to `~/.claude/hooks/archive/`. |
| Qwen cold-start routing to paid fallback silently | LOW | 1. Add warm-up probe to `qwen-exec.js`. 2. Set HTTP timeout to 180s. 3. Verify `actual_model` field in token log on next session. 4. Refund approximate cloud cost from session budget. |
| VRAM exhaustion causing slow inference | MEDIUM | 1. Add explicit `num_ctx: 32768` to all ollama calls. 2. Add keep_alive=0 between Forge tasks. 3. Add context budget enforcement in dispatcher. 4. Monitor with nvidia-smi during next Balanced profile session. |
| Feedback loop counter reset across sessions | MEDIUM | 1. Implement `phase-state.js` with persistent loop counters. 2. Audit all loop-capable phases (Judge, Forge checkpoint, Crucible) to read persisted counts. 3. For any phase currently at or past max loops, surface to human before next run. |
| Token costs show negative or zero values | MEDIUM | 1. Audit `pricing.js` for all nine provider functions. 2. Check Anthropic calls: `cache_read_input_tokens` should reduce cost, not negate it. 3. Check Qwen calls: map `prompt_eval_count`/`eval_count` to token fields. 4. Re-compute costs for the last 7 days from raw_usage field in token log. |
| plugin.json in wrong directory (components not discovered) | LOW | 1. Move `plugin.json` to `.claude-plugin/plugin.json`. 2. Move any components from inside `.claude-plugin/` to plugin root. 3. Run `claude --plugin-dir <path> --debug` and confirm manifest loading message. |
| Infinite Crucible->Forge loop (costs spiraling) | HIGH | 1. Kill the session immediately. 2. Read the two Crucible outputs to identify if the same issues recur. 3. If recurring: Forge has a systematic error — escalate to human architectural review. 4. If different: Crucible is finding new issues each pass — likely a scope problem in the blueprint. 5. Manually patch the identified issues. 6. Reset loop counter in state.json to 0 before retrying. |

---

## Pitfall-to-Phase Mapping

How roadmap phases should address these pitfalls.

| Pitfall | Prevention Phase | Verification |
|---------|------------------|--------------|
| Hook double-registration | Plugin scaffold phase | `claude plugin validate` passes; `claude --debug` shows single hook registration |
| Old hooks still firing | Hook consolidation phase | Token log shows no entries from deprecated hook names after consolidation |
| Qwen cold-start timeout | Qwen executor implementation phase | First Qwen call per session completes without cloud fallback; warm-up probe logged |
| VRAM exhaustion | Qwen executor + Forge checkpoint phase | nvidia-smi shows <20GB during 32K context Forge session; inference speed stays above 15 tok/s |
| Gemini 429 silent failure | Gemini executor implementation phase | 429-injection test passes; retry attempts logged; Judge phase completes |
| Loop counter not persisted | Phase state infrastructure phase | Kill + restart test: loop count survives session interruption |
| Silent fallback without notification | Dispatch implementation phase | Artificial unavailability test: fallback logged with both requested and actual model |
| Token logging broken across providers | Pricing infrastructure phase | Cost calculation tests for all nine models pass; no negative or zero costs |
| Wrong plugin.json location | Plugin scaffold phase | `claude --plugin-dir --debug` confirms manifest found; all components discovered |
| Biased feedback loop termination | Phase command implementation (Judge, Crucible) | Machine-readable markers in output files; loop trigger tests with ambiguous prose |
| Hardcoded paths in forked executors | Executor fork phase | `grep -rn "claude/hooks" executors/` returns no matches |

---

## Sources

- Claude Code Plugin Reference (official, 2026): https://code.claude.com/docs/en/plugins-reference
- Claude Code Plugin Issues — Duplicate hooks.json: https://github.com/affaan-m/everything-claude-code/issues/103
- Claude Code Plugin Issues — Plugin hooks never fire: https://github.com/anthropics/claude-code/issues/27398
- Claude Code Plugin Issues — Hook version path stale after update: https://github.com/anthropics/claude-code/issues/18517
- InsiderLLM — Qwen 3.5 local guide, GPU fit and context length: https://insiderllm.com/guides/qwen35-local-guide-which-model-fits-your-gpu/
- ollama Issue #9209 — timed out waiting for llama runner to start: https://github.com/ollama/ollama/issues/9209
- OpenClaw Issue #43946 — Configurable LLM request timeout, ollama cold-start silent fallback: https://github.com/openclaw/openclaw/issues/43946
- Markaicode — Ollama production debugging guide 2025: https://markaicode.com/ollama-production-debugging-guide/
- Gemini API Rate Limits — Google AI for Developers: https://ai.google.dev/gemini-api/docs/rate-limits
- Gemini API Rate Limits 2026 complete guide: https://yingtu.ai/en/blog/gemini-api-rate-limits-explained
- LoopJar — Agent orchestration feedback loops production lessons: https://loopjar.ai/blog/agent-orchestration-feedback-loop
- Anthropic — Building Effective Agents: https://www.anthropic.com/research/building-effective-agents
- Portkey — Tracking LLM token usage across providers: https://portkey.ai/blog/tracking-llm-token-usage-across-providers-teams-and-workloads/
- antfarm Issue #304 — Provider-aware usage normalization for cost tracking: https://github.com/snarktank/antfarm/issues/304
- DEV Community — Silent killers in Node.js (unhandledRejection): https://dev.to/silentwatcher_95/the-silent-killers-in-nodejs-uncaughtexception-and-unhandledrejection-1p9b
- Phase 10 Adversarial Review (this project, v2.0) — HIGH severity: MiniMax token cost semantic mismatch

---
*Pitfalls research for: Six-phase multi-model plugin pipeline added to existing three-model hook system (Seraphim v3.0)*
*Researched: 2026-04-04*
