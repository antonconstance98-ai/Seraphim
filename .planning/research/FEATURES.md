# Feature Landscape

**Domain:** Multi-model AI coding agent integration (Claude Code + Codex CLI)
**Researched:** 2026-04-02
**Overall confidence:** HIGH — anchored by official docs (Claude Code hooks, Codex CLI, codex-plugin-cc), PROJECT.md requirements, and model comparison research from 34 sources.

---

## Table Stakes

Features users expect. Missing = integration feels broken or incomplete.

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| **Codex CLI invocation from Claude Code hooks** | The official `codex-plugin-cc` (released 2026-03-30) establishes this as the standard pattern. Users expect any integration to use the existing Codex install via CLI, not reinvent the wheel. | Low | `codex exec` non-interactive mode pipes final result to stdout, `--json` flag streams JSONL events. Use PostToolUse or Stop hooks to trigger. |
| **Stop hook review gate** | The official plugin already ships `/codex:setup --enable-review-gate`. Any integration that doesn't provide a review gate will feel like a downgrade from what OpenAI ships for free. | Medium | Stop hook fires when Claude finishes responding; return `decision: "block"` to force Claude to address issues. Needs `stop_hook_active` guard to prevent infinite loops. |
| **AGENTS.md spec file for Codex** | Codex's native context mechanism is AGENTS.md. Not providing one means Codex operates without project context — hallucinations, wrong conventions, broken output. | Low | File lives at repo root. Standard Markdown, no special syntax. Required for all Codex invocations to be project-aware. |
| **Explicit Opus-stays-architect boundary** | Research confirms (HIGH confidence) that Opus at 42 tok/s + $15/$75 pricing must not be wasted on routine tasks. Users expect the integration to enforce this boundary, not leave it to the user to remember. | Low | Encoded in hooks, CLAUDE.md, and AGENTS.md. Must be a hard rule, not a soft recommendation. |
| **Cross-model code review** | The HubSpot Sidekick implementation achieved 80% engineer approval and 90% faster PR feedback using a judge-agent pattern. The community consensus (Reddit, HN, Chandler Nguyen's blog) identifies this as the primary value of having two models. Any integration that doesn't wire up review is missing the main reason to have Codex. | Medium | `/codex:adversarial-review` for adversarial mode; `/codex:review` for standard. Pattern: Claude implements, Codex reviews — or vice versa. |
| **Token usage logging per call** | At $15/$75 per million tokens for Opus, users need to verify the cost savings claim. The "done" definition in PROJECT.md is explicitly "prove cost savings vs Opus-only baseline." Without per-call logs, there's no evidence the integration works. | Medium | Log: model name, task type, tokens_in, tokens_out, cost, timestamp. Tools like claudetop and tokscale exist for Claude Code; similar pattern for Codex via `--json` output. |
| **Fallback when primary model rate-limits** | Operational resilience is an explicit community benefit of dual-model setups. If Codex rate-limits, the workflow should degrade gracefully to Opus, not crash. | Low | Encode fallback routing in hooks: `if codex fails → route to Opus`. This is a reliability requirement, not a nice-to-have. |
| **PreToolUse routing hook** | PreToolUse is the only Claude Code hook that can block actions before they execute. Without it, there's no way to intercept Claude's intent to route tasks appropriately. | Low | Uses Claude Code's hook system (23 lifecycle events available). `PreToolUse` fires before any tool call and can redirect or block. |

---

## Differentiators

Features that go beyond what the official plugin ships. These are the reasons to build a custom integration rather than just installing `codex-plugin-cc`.

| Feature | Value Proposition | Complexity | Notes |
|---------|-------------------|------------|-------|
| **GSD wave-level Codex injection** | GSD's wave-based execution model (independent tasks run in parallel waves) is a natural fit for parallel Codex dispatch. Each wave can route background validation tasks to Codex-mini while the primary agent works. No off-the-shelf plugin does this. | High | Requires modifying GSD plugin source. GSD uses `.planning/` directory state. Integration point: post-wave-execution hook that dispatches Codex validation before wave completion is confirmed. |
| **Superpowers parallel hypothesis testing via Codex-Spark** | Superpowers' `dispatching-parallel-agents` skill already supports parallel execution. Routing hypothesis-testing threads to Codex-Spark (1,000+ tok/s) instead of spawning more Opus subagents reduces cost by ~10x per hypothesis. | High | Requires modifying Superpowers skill source. Codex-Spark is only available to ChatGPT Pro subscribers — user has Plus ($20/mo), which may not include Spark. Verify access before building. |
| **Adaptive handoff spec generation** | The current state of the art is static AGENTS.md files. An adaptive system generates handoff specs tailored to task complexity: file-level detail for risky/complex tasks, feature-level for routine ones. This reduces Codex hallucinations on complex work and reduces overhead on simple work. | High | Opus generates the spec as part of phase planning. Spec format is AGENTS.md-compatible Markdown. Complexity classifier: file count + cross-module dependencies + risk keywords. |
| **Opus-Codex plan review loop (2-3 rounds)** | Research (Chandler Nguyen, Hacker News consensus) confirms 2-3 rounds of cross-model review "produces significantly better results than either model working alone." The official plugin only does single-pass review. A 2-3 round loop is a concrete quality improvement that's easy to measure. | Medium | Round 1: Opus generates plan. Round 2: Codex reviews, returns amendments. Round 3: Opus incorporates, Codex confirms. GSD's plan-before-execute structure already has the right insertion point. |
| **Session cost report vs Opus-only baseline** | Token tracking alone is table stakes. The differentiating feature is the comparative report: "This session cost $X. Opus-only equivalent would have cost $Y. Savings: $Z (N%)." This is the proof of value the user needs to justify the integration. | Medium | Requires storing a baseline cost estimate (task-type → average Opus cost mapping from the model comparison research). Report generated at session end via Stop hook or `/gsd:complete-milestone`. |
| **Background Codex validation during Claude execution** | Using Codex-mini as a continuous background validator (non-blocking) while Opus executes primary tasks. The validation result is available when Opus finishes, not as a blocking gate. This gives quality assurance without adding latency. | High | Implementation: PostToolUse hook spawns `codex exec` in background (`&`), writes result to `.planning/validation-queue/`. Opus checks the queue at natural stopping points. |
| **Rate-limit aware routing with cost guardrails** | Automatically detect Codex Plus subscription limits (33-168 messages/day) and downshift to API when CLI is exhausted. Log a daily spend warning at 80% of the $5 Codex target. | Medium | Requires tracking CLI call count per day. If count > threshold, switch to OpenAI API. Persist count in `.planning/codex-usage.json`. |
| **Cross-wave integration check via Codex 1M context** | GPT-5.4's native 1M context window (vs Opus's 200K standard) makes it better suited for whole-codebase consistency checks after multiple waves complete. Route cross-wave integration verification to Codex rather than Opus. | Medium | Triggered by GSD's `/gsd:verify` or wave completion event. Codex receives full diff since last milestone + AGENTS.md for context. |

---

## Anti-Features

Features to explicitly NOT build. These create more problems than they solve.

| Anti-Feature | Why Avoid | What to Do Instead |
|--------------|-----------|-------------------|
| **Real-time streaming between models** | PROJECT.md explicitly rules this out. Beyond scope, it adds latency (model A waits on model B's stream), complicates error handling, and provides no quality benefit over async handoff. | Async handoff: Opus generates plan → writes to file → Codex reads and executes → Codex writes result → Opus reads result. Clean separation, no streaming needed. |
| **Codex making architectural decisions autonomously** | Codex hallucinates APIs at higher rates than Opus (research finding, MEDIUM confidence), sticks closer to literal instructions, and degrades near max context. Letting it architect is how you get technically-correct-but-wrong designs. | Hard-code the rule in AGENTS.md: "Never propose architectural changes. Implement what Opus specifies. Flag uncertainty rather than guess." |
| **Gemini or other model support** | Out of scope per PROJECT.md. Adding a third model multiplies routing complexity, configuration surface, and debugging burden without clear value. OpenAI and Anthropic already have complementary strengths. | Keep it binary: Opus for reasoning, Codex for execution. Any ambiguous case defaults to Opus. |
| **Web UI or dashboard for cost tracking** | This is a terminal-first integration on a dedicated workstation. A web dashboard requires a running server, complicates setup, and moves away from the CLI-first philosophy. | CLI cost report printed at session end, written to `.planning/session-reports/YYYY-MM-DD.md`. Human-readable, no server needed. |
| **Universal auto-routing based on prompt analysis** | Automatic intent classification is fragile — keywords don't reliably distinguish "complex reasoning" from "clearly-defined execution." False positives send architectural work to Codex. Building a reliable classifier is a project in itself. | Use explicit routing points: specific GSD lifecycle events (post-plan, post-wave) and Superpowers skills trigger Codex. Human confirms routing at ambiguous points. |
| **Infinite review loops** | The official `codex-plugin-cc` warns that the review gate "may create extended loops and drain usage limits quickly." Uncapped loops are a budget and reliability risk. | Cap at 3 review rounds maximum. After round 3, escalate to human review. Encode the limit in the hook with a counter stored in `.planning/review-state.json`. |
| **Modifying Claude Code itself** | Explicitly out of scope. Any change to Claude Code's core binary breaks on the next Claude Code update, requires re-implementation, and voids compatibility guarantees. | Use only officially-supported extension points: hooks, skills, agents, plugins, CLAUDE.md, and `.claude/settings.json`. All changes are additive to the plugin layer. |
| **Separate Codex session management CLI** | Building a custom CLI wrapper around Codex adds a maintenance burden (update with every Codex CLI release) and duplicates what `codex exec` and `codex resume` already provide. | Invoke Codex CLI directly from hooks and skill scripts. Use `codex resume --last` for continuation, `codex exec` for non-interactive automation. |

---

## Feature Dependencies

```
AGENTS.md spec file
  → All Codex invocations (Codex operates without project context without it)

PreToolUse routing hook
  → Background Codex validation (hook triggers the dispatch)
  → Cross-wave integration check (hook triggers at wave boundary)

Token usage logging per call
  → Session cost report vs Opus-only baseline (baseline comparison needs per-call data)
  → Rate-limit aware routing (daily spend tracking needs call logs)

Codex CLI invocation from hooks (table stakes)
  → Stop hook review gate
  → Cross-model code review
  → Adaptive handoff spec generation
  → Opus-Codex plan review loop
  → Background Codex validation

GSD wave-level Codex injection
  → Cross-wave integration check (requires wave state from GSD)
  → Background validation during execution (uses GSD wave lifecycle)

Adaptive handoff spec generation
  → Opus-Codex plan review loop (spec is what Codex reviews)
  → Superpowers parallel hypothesis testing (spec scopes each hypothesis thread)
```

---

## MVP Recommendation

Build in this order to get to a working, valuable integration as fast as possible:

**Phase 1 — Foundation (must ship first, everything else depends on these):**
1. AGENTS.md spec file — zero Codex value without it
2. Codex CLI invocation from Claude Code hooks — the plumbing everything uses
3. Token usage logging per call — needed to prove value from day one

**Phase 2 — Core Value (what the integration actually does):**
4. Stop hook review gate — single most impactful quality feature
5. Cross-model code review (GSD + Superpowers) — the primary use case
6. Opus-Codex plan review loop (2-3 rounds) — quality multiplier on all plans

**Phase 3 — Differentiators (what makes this better than just installing codex-plugin-cc):**
7. Adaptive handoff spec generation — reduces Codex errors on complex tasks
8. Session cost report vs Opus-only baseline — proves the integration works
9. Background Codex validation during Claude execution — non-blocking quality assurance
10. GSD wave-level Codex injection — full GSD lifecycle integration

**Defer indefinitely:**
- Superpowers parallel hypothesis testing via Codex-Spark: requires ChatGPT Pro subscription ($200/mo), user has Plus ($20/mo). Verify Spark access before building. If unavailable, substitute GPT-5.4-mini.
- Rate-limit aware routing: implement only if hitting Plus limits in practice (33-168 messages/day is generous for this use case).

---

## Sources

- [openai/codex-plugin-cc — Official OpenAI Codex plugin for Claude Code](https://github.com/openai/codex-plugin-cc)
- [Claude Code Hooks Guide — Official Anthropic docs](https://code.claude.com/docs/en/hooks-guide) (HIGH confidence — official docs)
- [Codex CLI Features — Official OpenAI docs](https://developers.openai.com/codex/cli/features) (HIGH confidence — official docs)
- [Codex Non-Interactive Mode — Official OpenAI docs](https://developers.openai.com/codex/noninteractive) (HIGH confidence — official docs)
- [HubSpot Sidekick multi-model code review — InfoQ, March 2026](https://www.infoq.com/news/2026/03/hubspot-ai-code-review-agent/) (HIGH confidence — published case study with metrics)
- [Chandler Nguyen — Dual-wielding Claude Code + Codex GPT-5.4](https://chandlernguyen.com/blog/2026/03/13/codex-gpt-5-4-vs-claude-code-opus-4-6-dual-wielding-ai-coding-tools/) (MEDIUM confidence — practitioner blog, widely cited)
- [Hacker News GPT-5.4 discussion — cross-model review patterns](https://news.ycombinator.com/item?id=47265045) (MEDIUM confidence — community consensus)
- [Superpowers dispatching-parallel-agents skill](https://github.com/obra/superpowers/blob/main/skills/dispatching-parallel-agents/SKILL.md) (HIGH confidence — official plugin source)
- [Agentic Coding 2026 complete guide — halallens.no](https://halallens.no/en/blog/agentic-coding-in-2026-the-complete-guide-to-plugins-multi-model-orchestration-and-ai-agent-teams) (MEDIUM confidence — comprehensive guide)
- [SmartScope — codex-plugin-cc analysis](https://smartscope.blog/en/blog/codex-plugin-cc-openai-claude-code-2026/) (MEDIUM confidence — third-party analysis of official plugin)
- [docs/research/opus-vs-codex-model-comparison.md](../../docs/research/opus-vs-codex-model-comparison.md) — project's own research document compiled from 34 sources (HIGH confidence for routing decisions)
- [.planning/PROJECT.md](../PROJECT.md) — project requirements (canonical source for scope decisions)
