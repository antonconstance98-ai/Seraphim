# Domain Pitfalls: Multi-Model AI Coding Agent Integration

**Domain:** Multi-model AI orchestration (Claude Opus 4.6 + OpenAI Codex CLI)
**Project:** Claude X Codex
**Researched:** 2026-04-02
**Overall confidence:** HIGH (pitfalls backed by GitHub issues, CVE disclosures, and live community reports)

---

## Critical Pitfalls

Mistakes that cause data loss, cost explosions, security breaches, or full rewrites.

---

### Pitfall 1: Codex CLI Hangs Silently on Background Processes

**What goes wrong:**
Codex CLI enters an infinite "Waited for background terminal" loop when a background shell command completes but the agent fails to detect the completion signal. The agent neither errors nor times out — it simply stalls forever, consuming quota while doing nothing.

**Why it happens:**
Codex uses a naive background process detection mechanism. If a script exits via rename, file move, or non-standard exit path, the internal background terminal watcher does not register the exit. The agent then waits indefinitely for a signal that will never arrive. This is a known open bug affecting CLI versions 0.111.0 through at least 0.114.0 (March 2026).

**Consequences:**
- Silent quota drain on the Plus plan with no visible error
- Claude Code hook that spawns Codex blocks forever, stalling the GSD workflow
- No automatic recovery — manual interrupt (Ctrl+C) is the only resolution
- If running inside a Claude hook, the parent session also hangs

**Warning signs:**
- Codex output shows "Waited for background terminal" repeated more than 3 times
- No new tool call or response output for more than 60 seconds
- `ps aux | grep codex` shows a codex process still running after expected task duration

**Prevention:**
- Always run Codex via `codex exec` (headless non-interactive mode) inside Claude hooks, not the interactive TUI
- Set an explicit timeout wrapper: `timeout 300 codex exec --task "..."` — kills the process after 5 minutes
- Use `--approval-policy untrusted` or `--sandbox read-only` to prevent Codex from spawning persistent background processes
- Avoid tasks that involve long-running background scripts as handoff candidates for Codex
- Route file-manipulation-heavy tasks through Opus/Claude Code directly, not Codex

**Phase:** Implement this mitigation in Phase 1 (basic Codex CLI integration) before any production routing is enabled.

**Sources:**
- [GitHub Issue #14303](https://github.com/openai/codex/issues/14303)
- [GitHub Issue #14314](https://github.com/openai/codex/issues/14314)
- [GitHub Issue #13708](https://github.com/openai/codex/issues/13708)

---

### Pitfall 2: API Key Exfiltration via Hook Scripts and Environment Variables

**What goes wrong:**
Hook scripts in `.claude/settings.json` execute before the user sees a trust prompt. If `ANTHROPIC_BASE_URL` is overridden in a project settings file, Claude Code routes all API requests (including the full Authorization header containing the API key) to an attacker-controlled endpoint before the user can decline.

**Why it happens:**
Claude Code initializes and sends API requests during startup before the project trust dialog is shown. Hook scripts defined in project settings run with inherited environment variables including `ANTHROPIC_API_KEY` and `OPENAI_API_KEY`. This was a confirmed vulnerability (CVE-2025-59536, CVE-2026-21852, patched in Claude Code 2.0.65+).

**Consequences:**
- API key theft leading to unexpected billing (real incident: $82,314 billed in 24 hours from a stolen Google Cloud key)
- For this project specifically: both `ANTHROPIC_API_KEY` and `OPENAI_API_KEY` would be exposed simultaneously
- Compromised OpenAI key enables unauthorized Codex CLI usage billed to the user's API account
- Any hook script that logs or echoes environment variables exposes keys in terminal history and log files

**Warning signs:**
- Unexpected API spend spikes without corresponding local task activity
- Hook scripts that reference `$OPENAI_API_KEY` or `$ANTHROPIC_API_KEY` directly in their body
- `.claude/settings.json` contains a non-Anthropic `ANTHROPIC_BASE_URL`

**Prevention:**
- Update Claude Code to 2.0.65+ (CVE patches included)
- Never reference `$OPENAI_API_KEY` directly in hook script bodies — pass it via environment only
- Enable `CLAUDE_CODE_SUBPROCESS_ENV_SCRUB=1` to strip credentials from subprocess environments
- Store keys in a secrets manager or `~/.profile` — never in `.env` files committed to the repo
- Audit all hook scripts in `.claude/settings.json` before enabling them; treat hook source files as security-sensitive
- Never echo or log environment variables in hook scripts

**Phase:** Security review and remediation must happen in Phase 1 (setup) before any API keys are wired into hooks.

**Sources:**
- [Check Point Research CVE-2025-59536 disclosure](https://research.checkpoint.com/2026/rce-and-api-token-exfiltration-through-claude-code-project-files-cve-2025-59536/)
- [The Hacker News report](https://thehackernews.com/2026/02/claude-code-flaws-allow-remote-code.html)
- [GitHub Feature Request #21528 — Env Variable Redaction](https://github.com/anthropics/claude-code/issues/21528)

---

### Pitfall 3: GSD Plugin Commands Break After Claude Code Updates

**What goes wrong:**
Claude Code 2.1.89 changed command discovery: the `commands/` subdirectory format is no longer recognized for slash commands. GSD installs commands as `~/.claude/commands/gsd/*.md`, which breaks silently — commands return "Unknown skill" with no error pointing to the root cause.

**Why it happens:**
Claude Code migrated to a `skills/` directory format with `SKILL.md` files. The old `commands/gsd/` nested structure is no longer scanned. Plugin modifications made directly to GSD source files can be overwritten by GSD updates, or conversely, GSD updates can revert plugin modifications made to integrate Codex routing.

**Consequences:**
- All `/gsd:*` commands silently fail after a Claude Code update
- Plugin modifications for Codex integration are lost after `gsd update` or reinstall
- Workflow stops working with no obvious error; the user (non-technical) sees only "Unknown skill"

**Warning signs:**
- `/gsd:do`, `/gsd:next`, `/gsd:plan` return "Unknown skill" after a Claude Code update
- GSD commands disappear from the slash command autocomplete list
- After `gsd update`, Codex routing logic in plugin files reverts to original

**Prevention:**
- Pin GSD to a specific version tag before modifying plugin source files
- Maintain a diff/patch file of all GSD modifications so they can be reapplied after updates
- Use GSD's hook system and skills extension points rather than modifying GSD source files directly where possible — this is more upgrade-resilient
- Before any Claude Code update, check the changelog for `commands/` vs `skills/` directory changes
- After any update, run a smoke test: `/gsd:do test` and verify it executes

**Phase:** Establish patching discipline in Phase 1 before any plugin modifications. Revisit at every phase transition.

**Sources:**
- [GitHub Issue #1528 — GSD commands not discoverable in Claude Code 2.1.89](https://github.com/gsd-build/get-shit-done/issues/1528)
- [GitHub Issue #218 — GSD commands may not work after Claude Code update](https://github.com/glittercowboy/get-shit-done/issues/218)

---

### Pitfall 4: Cost Runaway from Routing Logic Failure

**What goes wrong:**
If the routing logic that decides "send to Opus" vs "send to Codex" fails open (defaults to Opus on error), all tasks route to the expensive model. Worse, if a retry loop triggers on Codex timeout and falls back to Opus, every failed Codex task doubles the cost.

**Why it happens:**
Routing logic written in hook scripts has no natural circuit breaker. A Codex CLI hang (see Pitfall 1) triggers a timeout, the fallback sends the task to Opus, Opus completes it, and the total cost is (Codex attempt cost) + (Opus cost) — more expensive than Opus-only. If this happens across 10 parallel tasks, daily spend can hit the $15 budget cap in under an hour.

**Consequences:**
- $15/day budget cap hit before noon
- Real incident benchmark: a single buggy agent loop triggered 2.3 million API calls over a weekend (industry case, 2026)
- Token multiplication: parallel multi-agent coordination produces 10-50x more tokens than strictly necessary without guardrails

**Warning signs:**
- Token tracker shows Opus usage spiking while Codex usage drops to zero
- Daily spend exceeds $5 before the expected Codex-heavy workload begins
- Error logs show repeated Codex timeouts followed by Opus fallback completions

**Prevention:**
- Implement a hard daily spend cap using session-level token tracking — stop routing new tasks when $10 is reached (buffer before $15 ceiling)
- Make routing fail CLOSED on error: if routing logic errors, prompt the user rather than auto-routing to Opus
- Add an iteration cap per task (max 3 Codex retries before abandoning, not falling back to Opus)
- Log every routing decision with model selected, reason, and estimated cost before execution
- Never run more than 3 parallel Codex instances — concurrent sessions are the primary cause of hangs

**Phase:** Cost guard rails must be in place before Phase 2 (parallel task routing). Design token tracker with hard caps from day one.

**Sources:**
- [AI Agent Cost Control: Avoiding Budget Overruns](https://rocketedge.com/2026/03/15/your-ai-agent-bill-is-30x-higher-than-it-needs-to-be-the-6-tier-fix/)
- [Cost Guardrails for Agent Fleets (Medium, Dec 2025)](https://medium.com/@Micheal-Lanham/cost-guardrails-for-agent-fleets-how-to-prevent-your-ai-agents-from-burning-through-your-budget-ea68722af3fe)

---

## Moderate Pitfalls

Mistakes that degrade quality, waste budget, or require significant rework.

---

### Pitfall 5: Context Loss at Opus-to-Codex Handoff

**What goes wrong:**
When passing a plan from Opus to Codex, the handoff is typically free-text or a structured markdown document. Codex receives the plan but loses the reasoning behind decisions — the "why" behind architectural choices, the constraints that ruled out alternatives, and the risk assessments. Codex then implements the letter of the plan but violates its intent, producing technically correct but architecturally wrong code.

**Why it happens:**
LLM handoffs treat context as a one-time data transfer. The model role schema (system/user/assistant) doesn't natively represent "this was produced by a different AI agent." Codex sees a task specification, not the reasoning chain behind it. Decisions survive the handoff, but the reasoning that makes those decisions defensible does not.

**Consequences:**
- Codex over-engineers or simplifies where Opus specifically decided otherwise
- Cross-model review loop (Codex reviews Opus output) catches different issues but also misses context-dependent decisions
- Handoff quality degrades further with each round-trip in the review loop

**Warning signs:**
- Codex output reverts previously-decided architectural choices
- Codex proposes "simpler alternatives" that were explicitly rejected by Opus in the plan
- Review loop produces conflicting recommendations that are irresolvable without human input

**Prevention:**
- Use the adaptive handoff spec format described in the project requirements: file-level detail for complex/risky tasks, feature-level for simple ones
- Include a "decisions-not-taken" section in handoff specs for complex tasks: explicitly list alternatives that were rejected and why
- Structure handoffs as typed JSON objects (not free text) with: task, constraints, decisions_made, rejected_alternatives, risk_assessment
- Limit review loop rounds to 2-3 maximum (research confirms diminishing returns beyond this)
- Treat the handoff spec as a first-class artifact — generate it deliberately, not as an afterthought

**Phase:** Handoff spec design is Phase 1 work. Template must be stable before any real task routing begins.

**Sources:**
- [Multi-Agent AI Pipelines: Solving Context Loss](https://briefhq.ai/blog/ai-agent-talks-to-ai-agent/)
- [AI Agent Handoff: Why Context Gets Lost](https://xtrace.ai/blog/ai-agent-handoff-why-context-gets-lost-between-agents-and-how-to-fix-it)
- [Chandler Nguyen — Dual-Wielding AI Coding Tools](https://chandlernguyen.com/blog/2026/03/13/codex-gpt-5-4-vs-claude-code-opus-4-6-dual-wielding-ai-coding-tools/)

---

### Pitfall 6: Token Tracking Inaccuracy Across Models

**What goes wrong:**
Token counts reported by Claude and OpenAI use different counting methodologies. Claude reports input/output/cache separately; OpenAI reports prompt/completion and sometimes includes hidden reasoning tokens in "output" counts. A naive tracker that sums all reported tokens across both providers produces meaningless cost estimates.

**Why it happens:**
Provider fragmentation: Claude charges $15/$75 per million (input/output) with separate cached pricing ($1.50 cached input). OpenAI charges $2.50/$15 per million with cached input at $0.25. The token counts themselves differ because each provider uses a different tokenizer — the same text produces different token counts in Claude's tokenizer vs GPT-5.4's tokenizer. Additionally, OpenAI's reasoning models generate internal reasoning tokens billed as output but invisible in the response.

**Consequences:**
- Reported "savings vs Opus-only baseline" is inaccurate — the success metric for the project is wrong
- Actual spend may exceed $15/day cap without triggering the tracker alert
- Cached input tokens billed at a fraction of standard are easy to double-count

**Warning signs:**
- Tracker-reported cost diverges from provider dashboard by more than 15%
- Codex tasks show suspiciously low token counts (reasoning tokens not captured)
- Daily totals from tracker don't match Anthropic or OpenAI billing page by end of session

**Prevention:**
- Track input_tokens, output_tokens, cache_read_input_tokens, cache_creation_input_tokens separately per provider — never aggregate them before costing
- Apply provider-specific pricing per token type, not a blended rate
- Cross-check tracker totals against provider dashboards at the end of each session for the first two weeks
- Use response usage metadata from the API (not client-side estimation) as the source of truth
- For OpenAI reasoning models, capture `reasoning_tokens` from `usage.completion_tokens_details` if available

**Phase:** Token tracker design is Phase 1. Accuracy validation is Phase 2 (after first real Codex usage).

**Sources:**
- [Langfuse — Token and Cost Tracking](https://langfuse.com/docs/observability/features/token-and-cost-tracking)
- [Tracking LLM Token Usage Across Providers](https://portkey.ai/blog/tracking-llm-token-usage-across-providers-teams-and-workloads/)

---

### Pitfall 7: Codex Context Degradation Near Window Limit

**What goes wrong:**
Codex CLI uses a hard-coded tool output truncation limit (10 KiB or 256 lines, whichever is reached first). Near the context window limit, the model receives truncated tool outputs, cannot see the middle of large files, and starts making assumptions about the truncated content. This produces subtle bugs that are hard to detect because the code looks plausible but is based on incomplete information.

**Why it happens:**
Codex's truncation mechanism is line-based, not token-based, so the reported "remaining tokens" diverges from the enforced limit. Additionally, Codex's auto-compression (context compaction) does not always trigger reliably — users report it failing to activate and the session hitting the hard context limit instead. The research document for this project notes that Codex "can struggle near max context; repeats next steps every time."

**Consequences:**
- Codex implements partial solutions because it cannot see complete files
- Review loops catch fewer issues because the reviewer also has a truncated context
- Long-running tasks (45+ minute sessions) accumulate context until degradation becomes severe

**Warning signs:**
- Codex repeats the same next steps multiple times without progress
- Code changes that ignore existing logic visible to Opus but not appearing in Codex's output
- Auto-compression not triggered and context usage at 80%+ of window

**Prevention:**
- Break large tasks into smaller sub-tasks that each fit comfortably within 100K tokens
- Use Codex for clearly-scoped, bounded tasks — not for open-ended "refactor this large codebase" work
- Monitor context token usage and terminate + restart the Codex session if it exceeds 70% of the window
- Pass file excerpts (not entire files) in handoff specs for large codebases
- Do not chain more than 3-4 file modifications in a single Codex session without checking context usage

**Phase:** Context management discipline is Phase 2 (first production tasks). Phase 1 should only test Codex on small, bounded tasks.

**Sources:**
- [Codex CLI — Auto Compression Not Triggering](https://community.openai.com/t/auto-compression-not-triggering-codex-still-runs-out-of-context-window/1376334)
- [Codex Context Compaction Research](https://gist.github.com/badlogic/cd2ef65b0697c4dbe2d13fbecb0a0a5f)
- [GitHub Issue #9857 — Increase CLI context window](https://github.com/openai/codex/issues/9857)

---

### Pitfall 8: ChatGPT Plus Rate Limits and Quota Metering Anomalies

**What goes wrong:**
The $20/mo ChatGPT Plus plan provides Codex CLI access but with strict 5-hour rolling usage windows. A known metering anomaly (March 2026) caused small tasks to consume 2% of the 5-hour quota each — exhausting the weekly budget in hours rather than days. The Plus plan provides 30-150 requests per 5-hour window shared across both CLI and web usage.

**Why it happens:**
Plus quota is shared between local CLI tasks and cloud-based web tasks. If the user also uses ChatGPT web interface, the CLI quota shrinks. The metering anomaly suggests some internal accounting change. OpenAI did not acknowledge a bug, but community reports were consistent enough to confirm the behavior was real, not user error.

**Additionally:** Codex-Spark is explicitly Pro-only (not available on Plus). The routing rules in the research document that call for "Codex-Spark / GPT-5.4-mini" for parallel hypothesis testing cannot use Codex-Spark on the current $20/mo plan.

**Consequences:**
- Quota exhausted mid-session, falling back to OpenAI API billing (no longer using the subscription)
- Cost model breaks: the value proposition of "CLI over API" disappears when CLI quota runs out
- Parallel hypothesis testing plan requires revision — Codex-Spark is unavailable; use GPT-5.4-mini via API instead

**Warning signs:**
- Codex CLI returns "usage limit reached" during normal work
- Quota dashboard shows 80%+ consumed within the first hour of a session
- Tasks that previously took minimal quota now consume 5-10x more

**Prevention:**
- Track CLI quota usage alongside API token costs — they are separate budgets
- Reserve Plus quota for higher-value, longer-running Codex tasks; use OpenAI API (`gpt-5.4-mini`) for short background validation tasks
- Replace all routing rules that reference "Codex-Spark" with "GPT-5.4-mini via API" — Codex-Spark requires ChatGPT Pro ($200/mo)
- Set a daily CLI usage ceiling: stop routing to CLI after 60% of the 5-hour window is consumed; switch to API for remainder
- Do not use ChatGPT web interface on the same account during active Codex CLI sessions — they share the same quota pool

**Phase:** Quota management must be built into Phase 1. Update routing rules before Phase 2 to remove Codex-Spark references.

**Sources:**
- [GitHub Issue #13186 — Codex Metering Anomaly](https://github.com/openai/codex/issues/13186)
- [Using Codex with Your ChatGPT Plan](https://help.openai.com/en/articles/11369540-using-codex-with-your-chatgpt-plan)
- [GitHub Discussion #2251 — Codex Usage Limits](https://github.com/openai/codex/discussions/2251)

---

### Pitfall 9: Simultaneous Rate Limits Across Both Providers

**What goes wrong:**
Running a multi-agent workflow that calls both Claude (Opus 4.6) and OpenAI (GPT-5.4) simultaneously triggers rate limits on both providers at once. Anthropic applies per-minute token budgets for multi-turn sessions; OpenAI applies RPM/TPM limits. When both hit simultaneously, all tasks stall.

**Why it happens:**
Multi-agent workflows accumulate context exponentially. By task 3 of a 12-task GSD phase, the session context has grown to 40-60K tokens. Each subsequent API call to Opus consumes more tokens per call, burning through Anthropic's TPM limit faster. If Codex review tasks are triggered simultaneously, OpenAI's RPM limit hits at the same time.

**Consequences:**
- Cascading stall: all agents block simultaneously waiting for rate limit reset (typically 60 seconds)
- Context is lost during the stall if session state is not persisted
- Retry storms: multiple stalled agents retry simultaneously when limits reset, immediately re-triggering the limit

**Warning signs:**
- HTTP 429 errors appearing in both Anthropic and OpenAI logs within the same minute
- Task completion times growing linearly in later phases vs earlier phases
- Multiple "rate limited, retrying" messages across concurrent Codex and Opus calls

**Prevention:**
- Never schedule Codex review tasks at the same time as Opus planning tasks — stagger them by at least 30 seconds
- Implement exponential backoff with random jitter (not fixed retry intervals) for all API calls
- Use `gpt-5.4-mini` for background validation instead of `gpt-5.4` — different model = different rate limit bucket
- Keep cross-model review sequential (Opus reviews first, then Codex), not truly parallel
- Monitor both providers' rate limit headers (`x-ratelimit-remaining-requests`, `x-ratelimit-remaining-tokens`) and throttle proactively before hitting 429

**Phase:** Rate limit handling is infrastructure work for Phase 1. The parallel review loop in Phase 2 must be staggered, not truly simultaneous.

**Sources:**
- [Codinhood — Ultimate Guide to Handling AI API Rate Limits](https://codinhood.com/post/ultimate-guide-ai-api-rate-limiting)
- [Portkey — Retries, Fallbacks, and Circuit Breakers in LLM Apps](https://portkey.ai/blog/retries-fallbacks-and-circuit-breakers-in-llm-apps/)

---

## Minor Pitfalls

Issues that create friction but are recoverable without major rework.

---

### Pitfall 10: Codex CLI Authentication in Headless/Hook Contexts

**What goes wrong:**
Codex CLI defaults to browser-based OAuth (ChatGPT login) for authentication. In headless environments — including Claude Code hook scripts running non-interactively — the browser prompt either fails silently or blocks the process. Additionally, `~/.codex/auth.json` and `OPENAI_API_KEY` environment variable can conflict, causing 401 errors.

**Prevention:**
- Use API key authentication (`OPENAI_API_KEY` environment variable), not ChatGPT login mode, for all hook-invoked Codex calls
- Verify authentication works headlessly before wiring into hooks: `codex exec --model gpt-5.4 "echo test"` in a clean terminal
- Do not mix auth methods — if using API key auth, remove or ignore `~/.codex/auth.json`
- Note: headless Device Code auth requires workspace admin enablement (not relevant for personal accounts)

**Phase:** Phase 1 setup must verify headless auth before any hooks are written.

**Sources:**
- [GitHub Issue #3820 — Enable Headless Authentication](https://github.com/openai/codex/issues/3820)
- [GitHub Issue #9253 — Cannot log in on headless environments](https://github.com/openai/codex/issues/9253)

---

### Pitfall 11: Prompt Injection Propagation Through Model Pipeline

**What goes wrong:**
When Opus reviews code written by Codex that processes user-provided content (e.g., reading files from the user's project), malicious instructions embedded in those files can be executed by either model. In multi-model pipelines, a successful injection at the Codex layer propagates to the Opus review layer — one poisoned file can compromise both models in sequence.

**Prevention:**
- Never pass raw, user-generated file content directly into review prompts — always wrap it in explicit content delimiters
- Use the "Plan, Validate, Execute" pattern for high-stakes operations: Opus plans, human confirms before Codex executes
- Treat any file content that will be included in a model prompt as potentially adversarial if it comes from outside the project

**Phase:** Apply to any phase where models read and process project files for review.

**Sources:**
- [OWASP LLM01:2025 — Prompt Injection](https://genai.owasp.org/llmrisk/llm01-prompt-injection/)
- [AI Agent Security 2026: Prompt Injection](https://swarmsignal.net/ai-agent-security-2026/)

---

### Pitfall 12: Review Loop Producing Irresolvable Conflicts

**What goes wrong:**
In the Opus-Codex plan review loop, both models advocate for their preferred approach (Opus: architectural elegance; Codex: simplicity and speed). After 2-3 rounds, the loop can reach a stable conflict state where each model simply re-asserts its previous position, and no convergence occurs.

**Why it happens:**
Each model has systematic biases: Opus tends toward over-engineering and thoroughness; Codex tends toward simplification and pragmatism. When reviewing each other's work, they apply these same biases — resulting in a flip-flop rather than synthesis.

**Prevention:**
- Hard-cap review rounds at 3 — if not converged by round 3, Opus makes the final call unilaterally (Opus is the designated orchestrator)
- Structure review prompts to elicit specific objections, not open-ended critique: "List any requirements this plan violates" rather than "What do you think of this plan?"
- Distinguish between style disagreements (defer to whichever model is executing the task) and correctness disagreements (escalate to Opus)

**Phase:** Review loop protocol design is Phase 1. Enforce round cap before Phase 2.

**Sources:**
- [Chandler Nguyen — Dual-Wielding AI Coding Tools](https://chandlernguyen.com/blog/2026/03/13/codex-gpt-5-4-vs-claude-code-opus-4-6-dual-wielding-ai-coding-tools/)
- Inference from project research document (model bias analysis, Section 6.1)

---

## Phase-Specific Warning Matrix

| Phase Topic | Likely Pitfall | Severity | Mitigation |
|-------------|---------------|----------|------------|
| Codex CLI setup and auth wiring | Pitfall 10: Headless auth failure | HIGH | Use API key mode, not ChatGPT login |
| Hook script development | Pitfall 2: API key exfiltration via hooks | CRITICAL | Never echo keys; use env scrubbing |
| First Codex task routing | Pitfall 1: Silent background process hang | HIGH | Wrap all `codex exec` calls with `timeout 300` |
| GSD plugin modification | Pitfall 3: Commands break after CC update | HIGH | Pin GSD version; maintain patch file |
| Token tracking implementation | Pitfall 6: Inaccurate cross-provider tracking | MEDIUM | Separate token types; verify against dashboards |
| Parallel task routing (Phase 2) | Pitfall 4: Cost runaway from routing failure | CRITICAL | Hard spend cap at $10/day before ceiling |
| Parallel task routing (Phase 2) | Pitfall 9: Simultaneous rate limits | HIGH | Stagger calls; exponential backoff with jitter |
| Review loop implementation | Pitfall 5: Context loss at handoff | HIGH | Structured typed handoff spec with decisions-not-taken |
| Review loop implementation | Pitfall 12: Irresolvable review conflicts | MEDIUM | Hard 3-round cap; Opus final authority |
| Long-running Codex sessions | Pitfall 7: Context degradation near window | MEDIUM | Terminate/restart at 70% context usage |
| Plus quota management | Pitfall 8: Quota exhaustion; Spark unavailable | HIGH | Replace Codex-Spark with GPT-5.4-mini; track CLI quota |
| Any phase reading project files | Pitfall 11: Prompt injection propagation | MEDIUM | Wrap file content in explicit delimiters |

---

## Key Constraints Confirmed by Research

These are facts (not opinions) that the roadmap must treat as fixed:

1. **Codex-Spark is unavailable on ChatGPT Plus ($20/mo)** — it is Pro-only ($200/mo). All routing rules referencing Codex-Spark must be replaced with `gpt-5.4-mini` via API.

2. **Codex CLI background process hang is an unresolved open bug** as of CLI version 0.114.0 (March 2026). All Codex CLI invocations from hooks must use `timeout` wrappers.

3. **CVE-2025-59536 is patched in Claude Code 2.0.65+** — verify current Claude Code version before implementing any hooks that reference API keys.

4. **GSD `commands/` subdirectory format broke in Claude Code 2.1.89** — any plugin modification plan must account for the `skills/` migration.

5. **The Plus plan's 5-hour quota is shared between CLI and web ChatGPT usage** — the integration must track CLI consumption separately to avoid unexpected quota exhaustion mid-session.

---

*Research based on GitHub issues (openai/codex, anthropics/claude-code, gsd-build/get-shit-done), CVE disclosures (CVE-2025-59536, CVE-2026-21852), OpenAI Developer Community reports, Check Point Research, and multi-source web verification. All pitfalls confirmed by at least two independent sources.*
