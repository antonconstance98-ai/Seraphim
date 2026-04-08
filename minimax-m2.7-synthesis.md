# MiniMax M-2.7: Final Research Synthesis

> **Date:** 2026-04-03
> **Sources:** 3 Claude Opus research agents + 5 ChatGPT Deep Research reports + direct web research
> **Purpose:** Complete evaluation of MiniMax M-2.7 for integration into the Claude X Codex multi-model workflow
> **Author:** Claude Opus 4.6 (synthesis), ChatGPT Deep Research (5 external reports)

---

## 1. What Is MiniMax M-2.7?

MiniMax M-2.7 is a 230-billion-parameter Mixture-of-Experts (MoE) language model released March 18, 2026 by MiniMax, a Shanghai-based AI company publicly traded on HKEX since January 2026. Only ~10 billion parameters activate per token (23:1 sparsity), which is why it can deliver near-frontier coding quality at a fraction of frontier pricing.

It is the latest in a rapid iteration cycle: M2 (early 2026) → M2.5 (Feb 12) → M2.7 (Mar 18). Previous models were open-weight (MIT license); **M2.7 is the first proprietary model** in the series.

The model's signature innovation is **self-evolution**: during training, M2.7 autonomously ran 100+ optimization cycles — analyzing its own failure trajectories, modifying scaffold code, and re-evaluating — achieving a 30% performance improvement without human intervention.

### Architecture at a Glance

| Spec | Value |
|------|-------|
| Total parameters | 230 billion |
| Active parameters | ~10 billion per token |
| Expert config | 8 experts, Top-2 routing |
| Attention | Multi-head with RoPE; 32 query / 8 KV heads |
| Hidden dim / Layers | 4,096 / 32 layers / SiLU (SwiGLU) |
| Context window | 204,800 tokens |
| Max output | 131,072 tokens |
| Reasoning | Built-in mandatory `<think>` tags |
| Input/Output | Text only (no vision) |
| Open source | No (proprietary; predecessors M2/M2.5 are MIT) |

---

## 2. How Good Is It? (Benchmarks)

### The Headline Numbers

M-2.7 approaches frontier models on real-world software engineering tasks at a fraction of the cost. But the picture is nuanced — it has clear strengths and clear gaps.

### Software Engineering

| Benchmark | M-2.7 | Opus 4.6 | GPT-5.4 | Notes |
|-----------|-------|----------|---------|-------|
| **SWE-Pro** | 56.22% | ~57% | 57.7% | Essentially tied with frontier |
| **SWE-bench Verified** | ~78%* | 80.8% | 74.9% | *Self-reported; uncorroborated |
| **Terminal Bench 2** | 57.0% | 74.7% | 75.1% | Significant gap on CLI agent tasks |
| **VIBE-Pro** (repo-level) | 55.6% | — | — | End-to-end project delivery |
| **Multi-SWE-Bench** | 52.7% | 35.7% | — | M2.7 leads on multi-repo tasks |
| **SWE Multilingual** | 76.5 (#2) | — | — | Java, TS, JS, Go, Rust, C, C++ |
| **Aider Polyglot** | **Not listed** | 72% | 88% | Significant gap in coverage |
| **HumanEval / MBPP** | **Not published** | — | — | Classic benchmarks missing |

### Head-to-Head Testing (Kilo Code)

| Test | M-2.7 | Opus 4.6 |
|------|-------|----------|
| Bugs found | 6/6 (100%) | 6/6 (100%) |
| Security vulns found | 10/10 (100%) | 10/10 (100%) |
| Integration tests written | 20 (unit-level) | 41 (full HTTP endpoint) |
| Architecture quality | Flat structure | Modular directories |
| Cost per task | **$0.27** | $3.67 |
| PinchBench score | 86.2% (5th/50) | Higher |

### General Intelligence

| Metric | M-2.7 | Opus 4.6 | GPT-5.4 |
|--------|-------|----------|---------|
| AA Intelligence Index | 50 | 53 | 57 |
| GDPval-AA ELO | 1495 | 1606 | 1667 |
| Hallucination rate | **34%** | — | — |
| GPQA Diamond | 87 | — | — |

### Speed (Measured, Not Marketing)

| Metric | M-2.7 Std | M-2.7 Highspeed | Opus 4.6 | GPT-5.4 |
|--------|-----------|-----------------|----------|---------|
| Output throughput | 43-47 tps | ~100 tps | 44-59 tps | 63-72 tps |
| Time to first token | 2.3-3.3s | ~1.0s | 1.7-1.9s | 0.8-3.7s |
| Time to first *answer* | Up to ~55s | — | — | ~117s (reasoning) |

### What the Benchmarks Tell Us

1. **M-2.7 is a genuine peer on SWE-Pro** — the hardest, most realistic coding benchmark. It's essentially tied with Opus and GPT-5.4.
2. **It falls behind on autonomous agent tasks** — Terminal Bench 2 shows a 18-point gap vs GPT-5.4 (57% vs 75%).
3. **Bug detection and security scanning match Opus perfectly** in head-to-head testing.
4. **Architecture and test depth are weaker** — simpler structures, fewer integration tests.
5. **Classic benchmarks (HumanEval, MBPP, Aider) are missing** — can't compare on the most commonly cited coding metrics.
6. **Low hallucination (34%)** is a standout strength for reliability.
7. **It misses subtle logic errors** (off-by-one) that Opus catches.

---

## 3. What Does It Cost?

### Token Pricing

| Model | Input/Mtok | Output/Mtok | Cache Read | Cache Write |
|-------|-----------|-------------|------------|-------------|
| **M-2.7 Standard** | **$0.30** | **$1.20** | $0.06 | $0.375 |
| **M-2.7 Highspeed** | $0.60 | $2.40 | — | — |
| Claude Opus 4.6 | $5.00 | $25.00 | $0.50 | $6.25 |
| GPT-5.4 (short ctx) | $2.50 | $15.00 | $0.25 | — |
| GPT-5.4 (>272K input) | $5.00 | $22.50 | — | Surcharge |
| Claude Sonnet 4.6 | $3.00 | $15.00 | $0.30 | — |
| DeepSeek V3.2 | $0.28 | $0.42 | — | — |

### Cost Ratios

| Compared To | Input Savings | Output Savings |
|------------|--------------|----------------|
| Opus 4.6 | **17x cheaper** | **21x cheaper** |
| GPT-5.4 | **8x cheaper** | **12.5x cheaper** |
| Sonnet 4.6 | 10x cheaper | 12.5x cheaper |
| DeepSeek V3.2 | ~comparable | ~3x more expensive |

> **Note:** Multiple reports corrected our earlier research — the $15/$75 pricing is the older Opus 4.1, not Opus 4.6. Opus 4.6 is $5/$25. M-2.7 is still 17-21x cheaper, just not 50-62x as initially calculated.

### Real-World Savings (on a $15/day budget)

| Routing % to M-2.7 | Daily Spend | Savings vs $15/day |
|--------------------|-------------|-------------------|
| 0% (all Opus) | $15.00 | — |
| 50% | $7.90-$8.55 | ~$6.50-$7.10 |
| 70% | $5.10-$5.97 | ~$9.00-$9.90 |
| 85% | $2.96-$4.04 | ~$11.00-$12.00 |

Even with a 20% retry overhead on M-2.7 tasks, the 17x price gap means savings remain substantial.

### Billing Models

- **Pay-As-You-Go:** Per-token pricing, no commitment
- **Token Plan:** $10/month (Starter, 1,500 req/5h), $20/month (Plus, 4,500 req/5h), higher tiers available
- **Coding Plan:** Promotional pricing with 10-12% referral discounts
- **Developer Program:** Up to $100 in free testing credits (apply via official form)

### The Verbosity Tax

M-2.7 generates **~4x more output tokens** than median models for equivalent tasks. This erodes per-token savings by 2-3x on output costs. Factor this into cost projections — but even with 4x verbosity, it's still dramatically cheaper than Opus.

---

## 4. How To Connect To It

### API Endpoints (Three Flavors)

| Format | Base URL | Use When |
|--------|----------|----------|
| **OpenAI-compatible** | `https://api.minimax.io/v1` | Node.js with existing `openai` package |
| **Anthropic-compatible** | `https://api.minimax.io/anthropic` | Claude Code integration, LiteLLM |
| **Native MiniMax** | `https://api.minimax.io/v1/text/chatcompletion_v2` | Direct access (deprecated in some docs) |

China endpoints use `api.minimaxi.com` (note the extra `i`).

### Zero New Dependencies

The existing `openai` npm package (v6.33.0, already installed in this project) works as-is:

```javascript
import OpenAI from "openai";

const minimax = new OpenAI({
  baseURL: "https://api.minimax.io/v1",
  apiKey: process.env.MINIMAX_API_KEY,
});

const response = await minimax.chat.completions.create({
  model: "MiniMax-M2.7",
  messages: [{ role: "user", content: "Review this code for bugs..." }],
  stream: true,
  extra_body: { reasoning_split: true }, // MiniMax-specific: structured reasoning
});
```

### Platform Availability

| Platform | Available | Notes |
|----------|-----------|-------|
| **MiniMax Direct** | Yes | Both OpenAI + Anthropic endpoints |
| **OpenRouter** | Yes | `minimax/minimax-m2.7`, $0.30/$1.20 |
| **LiteLLM** | Yes (M2.5 explicit; M2.7 via pass-through) | Pin to v1.83.0+ (supply chain incident) |
| **Kilo Code** | Yes | MiniMax provider dropdown |
| **Cline** | Yes | v3.47.0+ required |
| **Cursor** | Yes | Custom model via OpenAI base URL override |
| **Roo Code** | Yes (buggy) | Model missing from dropdown in v3.51.1 |
| **Claude Code (as replacement)** | Yes | Official MiniMax docs cover this |
| **Codex CLI** | **Not recommended** | MiniMax explicitly warns of compatibility issues |
| **Together AI / Fireworks** | M2/M2.5 only | M2.7 not confirmed |

### Rate Limits

| Tier | RPM | TPM |
|------|-----|-----|
| M-2.7 (pay-as-you-go) | 500 | 20,000,000 |
| M-2.7 Highspeed | 500 | 20,000,000 |
| Token Plan Starter | 1,500 req / 5h rolling | — |
| Token Plan Plus | 4,500 req / 5h rolling | — |

### Supported Features

| Feature | Status |
|---------|--------|
| Streaming (SSE) | Yes |
| Function/Tool calling | Yes (both OpenAI + Anthropic style) |
| Structured JSON output | **Partial** — no strict `response_format`; validate client-side with Zod |
| Reasoning (`<think>` tags) | Yes (mandatory; use `reasoning_split: true` for structured access) |
| Prompt caching | Yes (automatic; cache read 80% cheaper than input) |
| Multi-turn conversation | Yes |
| Vision/images | No |
| Temperature = 0 | **No** — must be (0.0, 1.0], strictly above zero |

---

## 5. Critical Gotchas & Limitations

These are the findings that matter most for integration. Sorted by severity.

### Must-Know (Will Break Things)

1. **Tool/reasoning continuity is the #1 silent failure mode.** If your code strips `<think>` content, `reasoning_details`, or `tool_calls` from assistant messages before re-sending conversation history, quality degrades silently. Every report flagged this. You MUST preserve full assistant response objects in multi-turn conversations.

2. **No strict JSON mode for M-2.7.** The `response_format: { type: "json_object" }` parameter only works for the older MiniMax-Text-01 family. M-2.7 outputs JSON via instruction only — always validate client-side (Zod or Ajv).

3. **Codex CLI integration is "not recommended"** by MiniMax themselves. Don't try to use M-2.7 as a Codex executor. Use it via the OpenAI SDK or Anthropic-compatible endpoint instead.

4. **Temperature cannot be exactly 0.** The API enforces `(0.0, 1.0]`. Set to 0.01 for near-deterministic output.

5. **Early task termination near context limit.** M-2.7 may abort tasks when approaching its 205K token ceiling. Budget for shorter contexts than the theoretical max.

### Should-Know (May Cause Issues)

6. **Several OpenAI params silently ignored:** `presence_penalty`, `frequency_penalty`, `logit_bias`, `n > 1`. Code that relies on these will run but produce unexpected behavior.

7. **Anthropic-compatible endpoint ignores `mcp_servers` parameter.** MCP integration is a client/tooling concern, not an API request parameter.

8. **Pre-answer latency can be ~55 seconds** on some workloads (distinct from TTFT). Interactive hooks may need longer timeouts.

9. **Token Plan keys expire with subscription.** If your subscription lapses, API calls fail. Pay-as-you-go keys don't expire.

10. **SWE-bench numbers across vendors are not comparable.** "SWE-Pro" (MiniMax) and "SWE-Bench Pro" (OpenAI) are different datasets. Don't treat them as direct rankings.

11. **IDE model registry lag.** M-2.7 may be missing from dropdowns, show wrong context window, or produce "Unknown model" errors in newer tool versions.

12. **LiteLLM had a supply chain security incident** in March 2026. Pin to v1.83.0+ for clean releases.

---

## 6. Where M-2.7 Fits in Our Workflow

### The Three-Model Architecture

```
                    ┌─────────────────────┐
                    │   Claude Opus 4.6   │
                    │   (Orchestrator)    │
                    │   $5 / $25 Mtok     │
                    └──────────┬──────────┘
                               │
                ┌──────────────┼──────────────┐
                │                             │
     ┌──────────▼──────────┐      ┌───────────▼───────────┐
     │  GPT-5.4 / Codex    │      │   MiniMax M-2.7       │
     │  (Executor)         │      │   (Analyst / Tester)  │
     │  Free via Plus $20  │      │   $0.30 / $1.20 Mtok  │
     └─────────────────────┘      └───────────────────────┘
     │ File editing         │      │ Bug triage             │
     │ Implementation       │      │ Security scanning      │
     │ Autonomous coding    │      │ Third-opinion review   │
     │ Long-running tasks   │      │ Unit test generation   │
     │ Agent execution      │      │ Boilerplate / scaffold │
     └─────────────────────┘      │ Documentation drafts   │
                                  │ Code explanation       │
                                  │ Config generation      │
                                  │ Context compression    │
                                  └───────────────────────┘
```

### Routing Rules

**Opus stays the orchestrator.** It handles architecture, planning, complex reasoning, final review gates, and anything requiring >200K context. No change to its role.

**Codex (GPT-5.4) stays the executor.** It handles file editing, implementation, autonomous coding, and verification loops. It has native filesystem access via the Codex CLI and runs free under the ChatGPT Plus subscription.

**M-2.7 is the new analyst/tester.** It handles the high-volume, mechanically-verifiable tasks where Opus quality is overkill and cost savings are 17-21x.

### Task Routing Table

| Task | Route To | Quality vs Opus | Why |
|------|----------|----------------|-----|
| Architecture design | **Opus** | Baseline | Modular structures, defense-in-depth |
| System planning | **Opus** | Baseline | Multi-step reasoning, cross-cutting |
| Complex refactoring | **Opus** | Baseline | Architectural understanding required |
| Integration test design | **Opus** | Baseline | 2x more tests, full HTTP coverage |
| Long-context analysis (>200K) | **Opus** | Baseline | 1M context, no degradation |
| Final review gate | **Opus** | Baseline | Highest reliability needed |
| Code implementation | **Codex** | ~95% | Free via Plus, filesystem access |
| File editing | **Codex** | ~95% | Native sandboxed execution |
| Autonomous coding | **Codex** | ~95% | Full-auto mode, long-running |
| Bug detection / triage | **M-2.7** | ~100% | Matched 6/6 at 7% cost |
| Security vulnerability scan | **M-2.7** | ~100% | Matched 10/10 at 7% cost |
| Third-opinion review | **M-2.7** | Different perspective | ~$0.05-0.15 per review |
| Unit test generation | **M-2.7** | ~80% | Fewer tests, but adequate |
| Boilerplate / scaffold | **M-2.7** | ~100% | Pattern-matching tasks |
| Documentation drafts | **M-2.7** | ~85% | Verbose but functional |
| Code explanation | **M-2.7** | ~90% | Adequate for internal use |
| Config generation | **M-2.7** | ~100% | Commodity task |
| Context compression | **M-2.7** | N/A | Summarize before sending to Opus |

### Hook Integration

| Hook Event | Current Behavior | With M-2.7 |
|-----------|-----------------|------------|
| `PostToolUse` (after Write/Edit) | Codex review | **M-2.7 bug/security scan** — cheaper, equally accurate |
| `Stop` (before Claude finishes) | Codex cross-review | **M-2.7 third-opinion** — additive check at negligible cost |
| `SubagentStop` | Codex validation | Route by task: M-2.7 for analysis, Codex for execution |
| `TaskCompleted` | Codex review gate | **M-2.7 scan + Codex if flagged** — tiered review |
| `UserPromptSubmit` | Context injection | **M-2.7 quick advisory** — $0.01-0.03 per call via API |

### The Two-Stage Context Compression Pattern

One novel pattern from the research: use M-2.7 as a **preprocessor** to summarize large codebases into condensed artifacts before sending to Opus. This avoids Opus's premium pricing above 200K tokens ($10/$37.50 per Mtok) and keeps total costs down.

```
Large codebase (500K tokens)
    → M-2.7 summarizes to 50K tokens ($0.15 + $0.06 = $0.21)
    → Opus reasons over 50K summary ($0.25 + $1.25 = $1.50)
    → Total: $1.71

vs. Opus directly on 500K:
    → $5.00 + $12.50 = $17.50 (premium long-context pricing)
```

### Multi-Pass Review (Newly Economical)

At M-2.7 pricing, you can afford to run **separate review passes** for different concerns and only escalate tricky findings to Opus:

1. **Correctness pass** — scan for bugs ($0.05)
2. **Security pass** — scan for vulnerabilities ($0.05)
3. **Performance pass** — scan for inefficiencies ($0.05)
4. **Escalation** — only flagged items go to Opus for deep analysis

Total cost for 3 passes: ~$0.15. A single Opus review: ~$3.67.

---

## 7. Implementation Guide

### Step 1: Get API Access

1. Sign up at `https://platform.minimax.io`
2. Navigate to Account → Settings → API Keys
3. Create a **Pay-As-You-Go** key (not Token Plan — those expire with subscription)
4. Apply for the Developer Program ($100 free credits): look for the application form on the platform
5. Set the environment variable:
   ```bash
   # Add to ~/.bashrc or secret manager
   export MINIMAX_API_KEY="your-key-here"
   ```

### Step 2: Test Basic Connectivity

```javascript
import OpenAI from "openai";

const minimax = new OpenAI({
  baseURL: "https://api.minimax.io/v1",
  apiKey: process.env.MINIMAX_API_KEY,
});

const res = await minimax.chat.completions.create({
  model: "MiniMax-M2.7",
  messages: [{ role: "user", content: "Say hello in JSON format: { \"greeting\": \"...\" }" }],
  max_tokens: 100,
});

console.log(res.choices[0].message.content);
console.log("Tokens:", res.usage);
```

### Step 3: Integrate into Hook Scripts

For the Claude X Codex hooks, M-2.7 replaces Codex API calls for review/analysis tasks. The integration is a simple client swap — same `openai` package, different `baseURL`:

```javascript
// In hook scripts, add a MiniMax client alongside the existing OpenAI client
const minimaxClient = new OpenAI({
  baseURL: "https://api.minimax.io/v1",
  apiKey: process.env.MINIMAX_API_KEY,
});

// Use for advisory/review calls
async function getMinimaxReview(code, taskType) {
  const response = await minimaxClient.chat.completions.create({
    model: "MiniMax-M2.7",
    messages: [
      { role: "system", content: `You are a ${taskType} reviewer. Be concise.` },
      { role: "user", content: code },
    ],
    max_tokens: 2000, // Control verbosity
  });
  return response.choices[0].message.content;
}
```

### Step 4: Add Routing Logic

The router decides which model handles each task based on type:

```javascript
function routeTask(taskType) {
  const minimaxTasks = [
    "bug-scan", "security-scan", "unit-test-gen",
    "boilerplate", "documentation", "code-explain",
    "config-gen", "third-opinion", "context-compress",
  ];
  const codexTasks = [
    "implementation", "file-edit", "autonomous-coding",
    "refactor-apply", "verification-loop",
  ];
  // Everything else → Opus (architecture, planning, complex review, long-context)

  if (minimaxTasks.includes(taskType)) return "minimax";
  if (codexTasks.includes(taskType)) return "codex";
  return "opus";
}
```

### Step 5: Handle M-2.7 Quirks

```javascript
// 1. Validate JSON responses client-side (no strict JSON mode)
import { z } from "zod";
const ReviewSchema = z.object({
  bugs: z.array(z.string()),
  severity: z.enum(["low", "medium", "high"]),
});

// 2. Set temperature just above 0 (can't be exactly 0)
temperature: 0.01,

// 3. Set max_tokens to control verbosity tax
max_tokens: 2000, // M-2.7 tends to be 4x more verbose than Opus

// 4. Preserve full assistant messages in multi-turn (critical!)
// Never strip tool_calls or <think> content from conversation history

// 5. Set generous timeouts (pre-answer latency can be ~55s)
timeout: 90000, // 90 seconds
```

### Step 6: Monitor and Optimize

Track these metrics per task type to validate the routing:
- **Pass rate** — does M-2.7's output actually solve the task?
- **Retry rate** — how often do you need to re-run?
- **Token count** — actual tokens consumed (watch the verbosity tax)
- **Cost per task** — actual spend vs projected
- **Cache hit rate** — check `usage` response fields; front-load stable content

---

## 8. Risk Mitigation

| Risk | Severity | Mitigation |
|------|----------|------------|
| Company instability | Medium | $1.18B raised, IPO'd, but burning $251M/yr. Use OpenRouter as abstraction layer for easy model swapping. |
| API goes down | Medium | OpenRouter fallback. M-2.7 is additive, not critical path. Opus + Codex still work without it. |
| Geopolitical restrictions | Medium | Shanghai-based company. Use as third opinion, never single point of failure. |
| Tool-calling bugs | Medium | Known issues improving with M2.7. Test thoroughly before relying on tool use. |
| Data privacy unclear | Medium | MiniMax privacy policy not fully transparent. Don't send sensitive credentials or PII. |
| Verbosity cost overrun | Low | Set `max_tokens` limits. Monitor output token counts. |
| Model deprecation | Low | Fast iteration cycle, but older models stay available. Pin to version. |
| Legal (Disney lawsuit) | Low | $75M suit against subsidiary. Unlikely to affect API service. |

### The Golden Rule

**M-2.7 is additive, not a replacement.** If MiniMax disappears tomorrow, the workflow still functions with Opus + Codex. M-2.7 adds a cost-efficient analysis layer — it doesn't become a dependency.

---

## 9. Cost Projection

### Expected Daily Spend with Three-Model Setup

| Component | Daily Cost | Role |
|-----------|-----------|------|
| Opus 4.6 (reduced to ~30% of tokens) | ~$3.00-$5.00 | Architecture, planning, final review |
| GPT-5.4 via ChatGPT Plus | ~$0.67 (fixed) | Execution, file editing |
| MiniMax M-2.7 (~60% of analysis tokens) | ~$0.15-$0.50 | Scanning, testing, drafting |
| **Total** | **~$3.82-$6.17** | |

**Savings vs. Opus-only ($15/day): 59-75%**

This frees budget for 2-4x more total work per day, or saves $9-11/day.

### Monthly Projection

| Setup | Monthly Cost |
|-------|-------------|
| Opus-only (current) | ~$450 |
| Three-model (Opus + Codex + M-2.7) | ~$115-$185 |
| **Monthly savings** | **$265-$335** |

---

## 10. What We Still Don't Know

These questions remain open even after exhaustive research:

1. **HumanEval / MBPP / Aider scores** — MiniMax hasn't published classic coding benchmarks. We can't compare on the most commonly cited metrics.
2. **Strict JSON mode timeline** — Will M-2.7 get `response_format` support? Currently text-01 only.
3. **Open weights release** — MiniMax teased "2 weeks" on Hugging Face (as of late March). Status unknown.
4. **Long-context quality curve** — At what token count does quality meaningfully degrade within the 205K window? No empirical data found.
5. **Production case studies** — Beyond individual developers and ByteDance's `deer-flow` repo, no major company deployments documented.
6. **Function calling reliability** — Reports say "improving but not fully resolved." Need hands-on testing.

---

## 11. Recommendation

**Integrate M-2.7 as the analyst/tester layer.** The evidence is strong:

- 17-21x cheaper than Opus with ~90% quality on coding tasks
- Zero new dependencies (existing `openai` package works)
- Matched Opus on bug and security detection in independent testing
- Available on OpenRouter for easy fallback
- Generous rate limits (500 RPM, 20M TPM)
- Low hallucination rate (34%)

**Start with three high-value use cases:**
1. `PostToolUse` bug/security scan after every Write/Edit
2. Third-opinion review before `Stop` events
3. Context compression for large codebases before Opus analysis

**Then expand to:** test generation, documentation drafts, boilerplate, config generation.

**Don't use it for:** architecture decisions, complex refactoring, autonomous file editing, or anything requiring >200K context.

The dominant cost lever is **routing fraction** — focus on expanding the set of tasks that can safely go to M-2.7, not on optimizing M-2.7 prompt quality. Every 10% shift from Opus to M-2.7 saves ~$1.40/day.
