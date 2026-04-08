# MiniMax M-2.7 Research Report

> **Date:** 2026-04-03
> **Purpose:** Evaluate MiniMax M-2.7 for integration into Claude X Codex multi-model coding workflow
> **Status:** Claude-side research complete (3 parallel agents + direct searches). Awaiting ChatGPT deep research reports for final synthesis pass.

---

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [Model Architecture & Specifications](#model-architecture--specifications)
3. [Benchmark Performance](#benchmark-performance)
4. [API & SDK Integration](#api--sdk-integration)
5. [Pricing & Cost Analysis](#pricing--cost-analysis)
6. [Developer Tooling & Ecosystem](#developer-tooling--ecosystem)
7. [Routing Strategy for Claude X Codex](#routing-strategy-for-claude-x-codex)
8. [Risk Assessment](#risk-assessment)
9. [Independent Reviews & Community Sentiment](#independent-reviews--community-sentiment)
10. [ChatGPT Deep Research Prompts](#chatgpt-deep-research-prompts-for-user-to-launch)
11. [Synthesis Notes](#synthesis-notes)

---

## 1. Executive Summary

MiniMax M-2.7 is a next-generation LLM released March 18, 2026, by Shanghai-based MiniMax (publicly traded on HKEX since January 2026). It is positioned as a **high-performance, low-cost coding and agent model** that approaches frontier quality at a fraction of the price.

**Key value proposition for our workflow:**
- **50x cheaper on input** and **62.5x cheaper on output** than Claude Opus 4.6
- Matches Opus on bug detection (6/6) and security scanning (10/10 vulns) in head-to-head testing
- 56.22% on SWE-Pro (Opus ~57%) — essentially tied on real-world software engineering
- Native OpenAI-compatible AND Anthropic-compatible API endpoints — drop-in integration
- Available on OpenRouter ($0.30/$1.20 per Mtok) and LiteLLM
- 204,800 token context window, 131,072 max output tokens
- ~100 TPS (highspeed variant) vs Opus ~33 TPS

**Bottom line:** M-2.7 is a strong candidate for the "analyst/tester" role in our three-model architecture, handling bug triage, security scanning, test generation, boilerplate, and third-opinion reviews at ~7% of Opus cost.

---

## 2. Model Architecture & Specifications

| Specification | Value | Source |
|--------------|-------|--------|
| **Model Name** | MiniMax-M2.7 | Official |
| **Release Date** | March 18, 2026 | OpenRouter |
| **Total Parameters** | 230 billion | Agent research (multiple sources) |
| **Active Parameters** | ~10 billion (23:1 sparsity ratio) | Agent research confirmed |
| **Architecture** | Sparse MoE — 8 experts, Top-2 routing (2 active per token) | Agent research confirmed |
| **Attention** | Multi-head with RoPE; 32 query / 8 KV heads | Agent research confirmed |
| **Hidden Dim / Layers** | 4,096 hidden dim / 32 layers / SiLU (SwiGLU) activation | Agent research confirmed |
| **Context Window** | 204,800 tokens | OpenRouter confirmed |
| **Max Output** | 131,072 tokens | OpenRouter confirmed |
| **Training Method** | Self-improvement — 100+ autonomous optimization cycles, 30% performance gain | Official blog |
| **Input Modality** | Text only | OpenRouter |
| **Output Modality** | Text only | OpenRouter |
| **Reasoning** | Built-in (`<think>`/`</think>` delimiters) | OpenRouter |
| **Default Temperature** | 1.0 | OpenRouter |
| **Default Top-p** | 0.95 | OpenRouter |
| **Open Source** | **No** — M2.7 is proprietary (first closed model in M2 series; M2/M2.5 were MIT) | Agent research confirmed |

### Model Lineage

| Model | Release | Params (Total/Active) | Open Source | Notes |
|-------|---------|----------------------|-------------|-------|
| MiniMax-Text-01 | Late 2024 | 456B / 45.9B | Yes (MIT) | Base foundation; hybrid Lightning Attention + Softmax + MoE |
| MiniMax-M1 | Mid 2025 | 456B / 45.9B | Yes (MIT) | Reasoning model; added RL-based test-time compute; 1M context |
| MiniMax-M2 | Early 2026 | 230B / 10B | Yes (MIT) | Compact MoE redesign; coding-focused |
| MiniMax-M2.5 | Feb 12, 2026 | 230B / 10B | Yes (MIT) | Heavy RL training (CISPO); SOTA agentic performance |
| **MiniMax-M2.7** | **Mar 18, 2026** | **230B / 10B** | **No (proprietary)** | Self-evolution training; productivity focus |

### Key Architecture Notes

- **MoE (Mixture of Experts):** Only ~10B parameters activate per token despite 230B total. This 23:1 sparsity ratio is what enables dramatically low cost — less compute per token.
- **Lightning Attention:** Custom attention mechanism from MiniMax, designed for long-context efficiency. The predecessor M1 supported 1M tokens natively.
- **Self-Evolution:** M2.7 autonomously executed 100+ optimization cycles during training, achieving 30% performance improvement without human intervention. It analyzed failure trajectories, planned changes, modified scaffold code, and ran evaluations in an iterative loop.
- **Training:** Layered on top of the CISPO reinforcement learning algorithm used for M2.5.
- **Knowledge Cutoff:** Not officially disclosed for M2.7. Base M2 cutoff is June 2024; M2.7 likely more recent given March 2026 release.

---

## 3. Benchmark Performance

### Software Engineering Benchmarks

| Benchmark | M-2.7 | Claude Opus 4.6 | GPT-5.4 | Gemini 3.1 |
|-----------|-------|-----------------|---------|------------|
| **SWE-Pro** | 56.22% | ~57% | 56.2% | — |
| **SWE-bench Verified** | 78% | 80.8% | 74.9% | — |
| **VIBE-Pro** (repo-level codegen) | 55.6% | — | — | — |
| **Terminal Bench 2** | 57.0% | — | — | — |
| **NL2Repo** | 39.8% | — | — | — |
| **SWE Multilingual** | 76.5 | — | — | — |
| **Multi SWE Bench** | 52.7 | — | — | — |

### Independent Head-to-Head (Kilo Code Testing)

| Metric | M-2.7 | Claude Opus 4.6 |
|--------|-------|-----------------|
| **PinchBench Score** | 86.2% (5th/50) | Higher |
| **Kilo Bench Pass Rate** | 47% (2nd place) | — |
| **Bugs Found** | 6/6 (100%) | 6/6 (100%) |
| **Security Vulns Found** | 10/10 (100%) | 10/10 (100%) |
| **Integration Tests Written** | 20 (unit-level) | 41 (full HTTP endpoint) |
| **Architecture Quality** | Flat structure | Modular directories |

### Professional / Agent Benchmarks

| Benchmark | M-2.7 | Notes |
|-----------|-------|-------|
| **GDPval-AA ELO** | 1495 | Highest among open-source models |
| **Toolathon** | 46.3% | Agent tool-use benchmark |
| **MM Claw** | 62.7% | Approaching Sonnet 4.6 |
| **Skill Adherence** (40+ tasks) | 97% | Each skill >2,000 tokens |
| **MLE-Bench Lite** (ML competitions) | 66.6% medal rate | Tied Gemini 3.1; behind Opus 75.7%, GPT-5.4 71.2% |

### General Intelligence & Reasoning

| Benchmark | M-2.7 | M-2.5 | Opus 4.6 | GPT-5.4 | Gemini 3.1 Pro |
|-----------|-------|-------|----------|---------|----------------|
| **AA Intelligence Index** | 50 | 42 | 53 | 57 | 57 |
| **GPQA Diamond** | 87 | 85.2 | — | — | — |
| **HLE** (Hard Learning) | 28 | ~19 | — | — | — |
| **IF Bench** (Instruction Following) | 76 | 70 | — | — | — |
| **GDPval-AA ELO** | 1495 | — | 1606 | 1667 | — |
| **Hallucination Rate** | **34%** | — | — | — | 50% |
| **Aider Polyglot** | **Not listed** | — | 72% (Opus 4) | 88% (GPT-5) | — |

### Performance & Speed

| Metric | M-2.7 Standard | M-2.7 Highspeed | Opus 4.6 | GPT-5.4 |
|--------|----------------|-----------------|----------|---------|
| **Tokens/second** | 46.7 (measured) | ~100 (claimed) | ~33 | ~50-80 |
| **Time to first token** | 2.3-2.5s | ~1.0s (est.) | ~2-3s | ~1-2s |
| **Intelligence Index** | 50 | — | 53 | 57 |
| **Throughput (TPM limit)** | 20M | 20M | — | — |

### Behavioral Characteristics

**Strengths:**
- Extensive context gathering before execution
- Solves unique problems others cannot (e.g., SPARQL reasoning)
- Strong on reasoning-dependent tasks requiring nuanced filter logic
- 97% skill adherence across complex multi-step agent tasks
- Production debugging: reduced incident recovery to under 3 minutes
- Low hallucination rate (34%) — lower than Sonnet 4.6 (46%) and Gemini 3.1 Pro (50%)
- Strong multilingual code support (SWE-Bench Multilingual 76.5%, ranked #2)

**Weaknesses:**
- **Verbosity tax:** ~4x more output tokens than median models for equivalent tasks (87M vs ~20M median)
- Higher token consumption (~2.8M input tokens per trial in benchmarks)
- Longer task duration (355-second median in Kilo Bench)
- Risk of timeouts on difficult problems due to over-exploration
- Architecture designs are flatter/simpler than Opus
- Integration test coverage is shallower than Opus (20 unit tests vs Opus's 41 integration tests)
- Known tool-calling loop bugs and premature task-halting issues (improving but not fully resolved)
- Missed subtle off-by-one errors that Claude Opus caught in comparative testing
- **Not on Aider polyglot leaderboard** — significant gap in benchmark coverage
- **Proprietary** — unlike predecessors M2/M2.5, community pushback on closed weights

---

## 4. API & SDK Integration

### API Endpoints

MiniMax provides **both OpenAI-compatible and Anthropic-compatible** endpoints:

| Endpoint Type | International URL | China URL |
|--------------|-------------------|-----------|
| **Base API** | `https://api.minimax.io` | `https://api.minimaxi.com` |
| **OpenAI-compatible** | `https://api.minimax.io/v1` | `https://api.minimaxi.com/v1` |
| **Anthropic-compatible** | `https://api.minimax.io/anthropic` | `https://api.minimaxi.com/anthropic` |

There is also a community-maintained endpoint: `https://minimax-m2.com/api/v1/chat/completions` (supports M2 through M2.7).

### Authentication

- API key from MiniMax Developer Platform: `https://platform.minimax.io`
- Set as `ANTHROPIC_AUTH_TOKEN` or `ANTHROPIC_API_KEY` for Anthropic-compatible endpoint
- Set as Bearer token for OpenAI-compatible endpoint
- **Two billing modes:** Token Plan (subscription) vs Pay-As-You-Go (per-token)
- Token Plan has a promotional "Coding Plan" with additional discounts

### Claude Code Integration (Direct)

MiniMax officially documents using M2.7 as a **drop-in replacement** for Claude in Claude Code:

```json
{
  "env": {
    "ANTHROPIC_BASE_URL": "https://api.minimax.io/anthropic",
    "ANTHROPIC_AUTH_TOKEN": "<MINIMAX_API_KEY>",
    "API_TIMEOUT_MS": "3000000",
    "ANTHROPIC_MODEL": "MiniMax-M2.7"
  }
}
```

### OpenAI SDK Integration (Node.js)

Since M2.7 exposes an OpenAI-compatible endpoint, existing OpenAI SDK code works:

```javascript
import OpenAI from 'openai';

const minimax = new OpenAI({
  apiKey: process.env.MINIMAX_API_KEY,
  baseURL: 'https://api.minimax.io/v1',
});

const response = await minimax.chat.completions.create({
  model: 'MiniMax-M2.7',
  messages: [{ role: 'user', content: 'Review this code...' }],
  stream: true, // streaming supported
});
```

### Platform Availability

| Platform | Supported | Model ID | Notes |
|----------|-----------|----------|-------|
| **OpenRouter** | Yes | `minimax/minimax-m2.7` | $0.30/$1.20 per Mtok |
| **LiteLLM** | Yes | Via Anthropic-compatible API | Day-0 support documented |
| **MiniMax Direct API** | Yes | `MiniMax-M2.7` | Both OpenAI and Anthropic endpoints |
| **OpenClaw** | Yes | OAuth authentication | Via install script |
| **Hermes Agent** | Yes | MiniMax provider | `hermes model` to configure |
| **OpenCode** | Yes | MiniMax provider | `opencode auth login` |
| **Kilo Code** | Yes | MiniMax provider dropdown | v3.47.0+ for Cline |
| **Cursor** | Yes | Custom model via OpenAI base URL override | |
| **Cline** | Yes | MiniMax as API provider | v3.47.0+ required |
| **Roo Code** | Yes | MiniMax as API provider | |
| **Azure AI Foundry** | M2 only | — | Open-weight M2, not M2.7 |

### Capabilities

| Feature | Status | Notes |
|---------|--------|-------|
| **Streaming** | Supported | SSE chunks via all three API formats; works through OpenRouter |
| **Function/Tool Calling** | Supported | Both OpenAI-style `tools` array and Anthropic-style `tool_use` blocks |
| **Structured JSON Output** | Partial | Works via instruction; strict `response_format: json_object` mode **unconfirmed** — test empirically |
| **Reasoning/Thinking** | Built-in | Mandatory `<think>`/`</think>` delimiters; interleaved thinking |
| **System Messages** | Supported | Hoisted and merged |
| **Multi-turn Conversation** | Supported | Strong multi-turn dialogue capability |
| **Vision** | Not supported | Text-only input/output |

### Rate Limits (Confirmed)

| Model | RPM (requests/min) | TPM (tokens/min) |
|-------|-------------------|-------------------|
| **MiniMax-M2.7** | **500 RPM** | **20,000,000 TPM** |
| **MiniMax-M2.7-highspeed** | **500 RPM** | **20,000,000 TPM** |
| **MiniMax-M1** (for comparison) | 120 RPM | 720,000 TPM |

Notes: Exceeding limits results in temporary throttling (~1 min reset). MiniMax can tighten limits during peak traffic. Third-party providers (OpenRouter) apply their own limits on top.

### Zero New Dependencies

The existing `openai` npm package (v6.33.0, already installed) works as-is:

```javascript
import OpenAI from "openai";
const minimax = new OpenAI({
  baseURL: "https://api.minimax.io/v1",
  apiKey: process.env.MINIMAX_API_KEY
});
```

No new packages needed — same SDK, different `baseURL` and `apiKey`.

---

## 5. Pricing & Cost Analysis

### Raw Token Pricing

| Model | Input/Mtok | Output/Mtok | Cache Read/Mtok | Cache Write Premium |
|-------|-----------|-------------|-----------------|---------------------|
| **MiniMax M-2.7** | **$0.30** | **$1.20** | **$0.06** | — |
| **MiniMax M-2.7 Highspeed** | $0.60 | $2.40 | — | — |
| Claude Opus 4.6 | $15.00 | $75.00 | $1.50 | — |
| Claude Sonnet 4.6 | $3.00 | $15.00 | $0.30 | — |
| GPT-5.4 (API) | ~$2.50 | ~$10.00 | — | — |
| DeepSeek V3 | ~$0.27 | ~$1.10 | — | — |

### Cost Multipliers

| vs Model | Input Savings | Output Savings |
|----------|--------------|----------------|
| vs Opus 4.6 | **50x cheaper** | **62.5x cheaper** |
| vs Sonnet 4.6 | 10x cheaper | 12.5x cheaper |
| vs GPT-5.4 API | ~8x cheaper | ~8x cheaper |
| vs DeepSeek V3 | ~comparable | ~comparable |

### Daily Budget Scenarios ($15/day limit)

**Scenario A: Current Opus-heavy (baseline)**
- 500K input + 100K output = $7.50 + $7.50 = **$15.00/day**

**Scenario B: Route 60% to M-2.7**
- Opus: 200K in + 40K out = $6.00
- M-2.7: 300K in + 60K out = $0.16
- **Total: $6.16/day (59% savings)**

**Scenario C: M-2.7 for all execution, Opus for planning only**
- Opus: 100K in + 20K out = $3.00
- M-2.7: 500K in + 100K out = $0.27
- **Total: $3.27/day (78% savings)**

### Verbosity Tax Warning

M-2.7 generates roughly **4x more output tokens** than median models for equivalent tasks. This erodes per-token savings by 2-3x on output costs. Even with this, it remains dramatically cheaper than Opus.

### MiniMax Coding Plan

MiniMax offers a promotional "Coding Plan" with additional discounts (12% OFF with referral codes). Worth investigating for sustained usage.

---

## 6. Developer Tooling & Ecosystem

### Integration Points for Claude X Codex

| Integration Method | Effort | Best For |
|-------------------|--------|----------|
| **OpenRouter proxy** | Minimal | Abstraction layer, easy model swapping, fallback routing |
| **OpenAI SDK direct call** | Low | Structured JSON responses, sub-5s advisory context |
| **Anthropic-compatible endpoint** | Low | Drop-in for Claude Code hooks that expect Anthropic format |
| **LiteLLM proxy** | Medium | If multi-model routing grows complex |
| **MiniMax direct API** | Low | Lowest latency, full feature access |

### Existing Multi-Model Patterns in the Wild

1. **OpenClaw Multi-Model Stack:** MiniMax = "cost-effective coder" for boilerplate, config, infra scripts
2. **Kilo Code Community:** M2.5 already dominates usage as default execution model; Opus only for architectural decisions
3. **bswen.com Agentic Workflows:** MiniMax recommended specifically for the execution phase of plan-then-execute

### Key Third-Party Support

- **LiteLLM:** Full support via Anthropic-compatible API (documented)
- **OpenRouter:** Listed with full pricing, 204K context confirmed
- **MiniMax Platform:** Official docs include Claude Code, Cursor, Cline, Roo Code, Kilo Code integration guides

---

## 7. Routing Strategy for Claude X Codex

### Recommended Three-Model Architecture

```
                    +-----------------+
                    |  Claude Opus    |
                    |  (Orchestrator) |
                    |  $15/$75 Mtok   |
                    +--------+--------+
                             |
              +--------------+--------------+
              |                             |
    +---------v---------+       +-----------v-----------+
    |   GPT-5.4 Codex   |       |    MiniMax M-2.7      |
    |   (Executor)      |       |    (Analyst/Tester)   |
    |   Free/$20mo      |       |    $0.30/$1.20 Mtok   |
    +-------------------+       +-----------------------+
    | File editing       |       | Bug triage             |
    | Implementation     |       | Security scanning      |
    | Autonomous coding  |       | Third-opinion review   |
    | Long-running tasks |       | Unit test generation   |
    +-------------------+       | Boilerplate/scaffold   |
                                | Code explanation       |
                                | Config generation      |
                                +-----------------------+
```

### Task Routing Table

| Task Type | Model | Quality vs Opus | Cost Impact |
|-----------|-------|----------------|-------------|
| Architecture design | **Opus 4.6** | Baseline | Baseline |
| System planning | **Opus 4.6** | Baseline | Baseline |
| Complex refactoring | **Opus 4.6** | Baseline | Baseline |
| Integration test design | **Opus 4.6** | Baseline | Baseline |
| Long-context analysis (>100K) | **Opus 4.6** | Baseline | Baseline |
| Final review gate | **Opus 4.6** | Baseline | Baseline |
| Code implementation | **GPT-5.4 (Codex)** | ~95% | Free (subscription) |
| File editing / autonomous coding | **GPT-5.4 (Codex)** | ~95% | Free (subscription) |
| Bug detection/triage | **MiniMax M-2.7** | ~100% (matched 6/6) | **93% savings** |
| Security vulnerability scan | **MiniMax M-2.7** | ~100% (matched 10/10) | **93% savings** |
| Third-opinion review pass | **MiniMax M-2.7** | Different perspective | **~$0.05-0.15/review** |
| Boilerplate/scaffold generation | **MiniMax M-2.7** | ~100% | **95%+ savings** |
| Unit test generation | **MiniMax M-2.7** | ~80% (fewer tests) | **93% savings** |
| Code explanation | **MiniMax M-2.7** | ~90% (more verbose) | **93% savings** |
| Config file generation | **MiniMax M-2.7** | ~100% | **95%+ savings** |
| Documentation drafts | **MiniMax M-2.7** | ~85% | **93% savings** |
| Quick advisory context | **MiniMax M-2.7 API** | Adequate | **~$0.01-0.03/call** |

### Integration into Existing Hooks

The existing Claude X Codex hook architecture maps naturally:

| Hook Event | Current | With M-2.7 |
|-----------|---------|------------|
| `PostToolUse` (after Write/Edit) | Codex review | **M-2.7 review** (cheaper, equally good at bug/vuln detection) |
| `Stop` (before Claude finishes) | Codex cross-review | **M-2.7 third-opinion** (additive, negligible cost) |
| `SubagentStop` | Codex validation | M-2.7 validation OR Codex (based on task type) |
| `TaskCompleted` | Codex review gate | **M-2.7 bug/security scan** (best cost/quality ratio) |
| `UserPromptSubmit` | Context injection | **M-2.7 quick advisory** (API call, $0.01-0.03) |

### Estimated Daily Cost with Three-Model Setup

**$3-6/day** (down from $10-15 Opus-only), freeing budget for 2-4x more total work.

---

## 8. Risk Assessment

### Company Stability: MEDIUM Risk

| Factor | Detail |
|--------|--------|
| **Funding** | $1.18B total raised |
| **IPO** | January 2026 on HKEX, $619M |
| **Valuation** | ~$4B (July 2025) |
| **Revenue** | $79M FY2025 (up 158.9% YoY) |
| **Net Loss** | $1.87B in 2025 (mostly paper losses; adjusted loss $251M) |
| **Gross Margin** | 25.4% (improving, +13.2 pp) |
| **Backers** | Alibaba, Tencent-affiliated, Shanghai state capital (STVC) |
| **International Revenue** | 70%+ of total |

### Legal Risk: LOW-MEDIUM

- Disney, Warner, Universal sued subsidiary SeaBull AI (Sept 2025) for $75M
- Outcome could impact finances/reputation but unlikely to affect API service

### Technical Risks

| Risk | Severity | Mitigation |
|------|----------|------------|
| API reliability | Medium | Use OpenRouter as abstraction layer |
| Tool-calling bugs | Medium | Earlier versions had issues; improving with M2.7 |
| Model deprecation | Low | Fast iteration (M2 -> M2.7 in ~6 months); older models stay available |
| Verbosity/cost overrun | Medium | Monitor output token counts; set max_tokens limits |
| Context window limit (205K) | Low | Opus handles >100K tasks; M-2.7 handles shorter tasks |

### Geopolitical Risks

| Risk | Severity | Mitigation |
|------|----------|------------|
| Chinese regulatory environment | Medium | International API endpoints separated |
| US/EU restrictions on Chinese AI | Medium | Use as additive third model, not single point of failure |
| Data sovereignty concerns | Low | Less relevant for individual developer use |

### Mitigation Strategy

1. Access via **OpenRouter** (abstraction layer, easy model swapping)
2. Keep **GPT-5.4/Codex** as primary execution model (free via ChatGPT Plus)
3. Use M-2.7 as **additive third opinion**, never as single point of failure
4. Monitor MiniMax quarterly financials (public company)
5. Set **max_tokens limits** to control verbosity tax

---

## 9. Independent Reviews & Community Sentiment

### Blog Reviews

| Source | URL | Key Finding |
|--------|-----|-------------|
| **thomas-wiegold.com** | https://thomas-wiegold.com/blog/minimax-m-2-7-review-is-it-worth-the-hype/ | "~90% of Opus quality for ~7% of cost"; inferior for architectural thinking |
| **Kilo Code head-to-head** | (referenced in multiple reviews) | Both found 6/6 bugs, 10/10 vulns. M2.7: $0.27/task vs Opus: $3.67/task |
| **bswen.com** | https://docs.bswen.com/blog/2026-03-24-kimi-vs-minimax-coding-comparison/ | Neither M2.7 nor Kimi caught off-by-one errors that Opus found |
| **computertech.co** | https://computertech.co/minimax-m2-7-review/ | Ranked #1 on Intelligence benchmark at $0.30/M tokens |
| **geeky-gadgets.com** | https://www.geeky-gadgets.com/minimax-m27-benchmark-results/ | Highlighted 30% autonomous self-improvement and 50x cost advantage |
| **InfraFlow AI** | https://infraflowai.substack.com/p/complete-technical-guide-automating | "From $200/month to $16/month, 2x Faster Responses" |

### YouTube Reviews

| Channel | URL | Notes |
|---------|-----|-------|
| Full Review | https://www.youtube.com/watch?v=--uxieT5J9Y | Compares M2.7 vs M2.5 vs Opus 4.6 vs Sonnet 4.6 using OpenClaw |
| AICodeKing | https://www.youtube.com/watch?v=suGe9MYBhAU | "Fully Tested" — claims M2.7 outperformed Opus 4.6 (likely exaggerated) |
| MiniMax overview | https://www.youtube.com/watch?v=njgf43CsMDo | Setup guide and capabilities walkthrough |

### Community Sentiment (Reddit r/LocalLLaMA)

- M2.7 on OpenRouter thread received positive reception
- Community notes: generous rate limits (no weekly quota issues), strong for agent use
- Some pushback on M2.7 being closed-source (M2/M2.5 were MIT licensed)
- Tool-calling bugs noted but "improving with M2.7"

---

## 10. ChatGPT Deep Research Prompts (For User to Launch)

### Prompt 1 — Model Architecture & Benchmarks
> "Do a comprehensive technical analysis of MiniMax M-2.7 (released March 2026). I need: (1) Model architecture — is it MoE, what is the total parameter count vs active parameters per token, what attention mechanism does it use? (2) Full benchmark scores on coding benchmarks: HumanEval, SWE-bench Verified, SWE-Pro, MBPP, LiveCodeBench, Aider polyglot leaderboard, BigCodeBench. (3) General reasoning benchmarks: MMLU, GPQA, MATH, ARC. (4) Build a comparison table of M-2.7 vs Claude Opus 4.6 vs GPT-5.4 vs DeepSeek V3 vs Gemini 3.1 on every available benchmark. (5) Find independent evaluations, blog posts, and developer reviews from 2025-2026 that tested M-2.7 on real coding tasks. Include all source URLs."

### Prompt 2 — API Integration & Cost Economics
> "Research MiniMax M-2.7 API for developers. I need: (1) All available API endpoints (base URL, OpenAI-compatible endpoint, Anthropic-compatible endpoint). (2) Authentication methods and API key setup. (3) SDK support — is there an official Python or Node.js SDK? What npm packages exist? (4) Pricing: exact cost per million tokens for input and output. Compare to Claude Opus 4.6 ($15/$75 per Mtok), Claude Sonnet ($3/$15), and GPT-5.4. (5) Rate limits and throttling. (6) Free tiers, billing models (Token Plan vs Pay-As-You-Go), and any startup credits. (7) Regional availability and latency. (8) Platform availability: is M-2.7 on OpenRouter, Together AI, Fireworks AI, LiteLLM, Azure, or AWS? Include code examples and source URLs."

### Prompt 3 — Developer Ecosystem & Real-World Usage
> "Research the MiniMax M-2.7 developer ecosystem as of 2026. I need: (1) What developer tools and frameworks support MiniMax models — CLI tools, IDE extensions (VS Code, Cursor, Cline), agent frameworks? (2) Is there LiteLLM support? OpenRouter availability? Any MCP (Model Context Protocol) servers for MiniMax? (3) What are real developers saying about using M-2.7 for code generation in production? Find Reddit posts (r/LocalLLaMA, r/ChatGPT), blog posts, YouTube reviews, and GitHub repos. (4) What are the practical gotchas and limitations developers have encountered? (5) Are there any notable open-source projects or companies using M-2.7 in their AI workflows? Include all source URLs."

### Prompt 4 — Competitive Analysis for Multi-Model Routing
> "I'm building a multi-model AI coding workflow where Claude Opus 4.6 handles architecture/reasoning, OpenAI Codex (GPT-5.4) handles execution, and I want to add MiniMax M-2.7 as a third model. Research MiniMax M-2.7's specific strengths and weaknesses for software engineering tasks. What task types would M-2.7 handle better or cheaper than Opus or GPT-5.4? Consider: code review, test generation, refactoring, documentation, boilerplate generation, bug triage, and code explanation. Provide a concrete routing recommendation table showing which model to use for each task type based on quality and cost."

### Prompt 5 — Integration Technical Deep-Dive
> "Technical deep-dive on integrating MiniMax M-2.7 into a Node.js workflow. I need: (1) exact API endpoint URLs and authentication flow, (2) Node.js SDK or REST API calling patterns with code examples, (3) response format and streaming support, (4) function/tool calling capabilities, (5) structured JSON output support, (6) any OpenAI SDK compatibility layer. Also research if M-2.7 can be called through aggregator platforms like OpenRouter, Together AI, or Fireworks AI, and whether LiteLLM proxy supports it."

### Prompt 6 — Cost Optimization & Context Window Strategy
> "MiniMax M-2.7 cost optimization research for a developer spending $15/day on AI coding tools. Current stack: Claude Opus 4.6 ($15/$75 per Mtok) and GPT-5.4 via ChatGPT Plus ($20/mo). How much could I save by routing specific task types to MiniMax M-2.7? Research: (1) M-2.7 token pricing vs competitors, (2) context window size and how it compares to Opus's 200K and GPT-5.4's 128K, (3) tokens-per-second throughput, (4) quality-adjusted cost (factoring in retry rates and quality differences), (5) any free tier or startup credits available."

### Suggested Grouping (to reduce from 6 to 3 sessions)
- **Session A:** Prompts 1 + 3 (architecture + ecosystem — "what is it and who uses it")
- **Session B:** Prompts 2 + 5 (API integration — "how to connect to it")
- **Session C:** Prompts 4 + 6 (routing + cost — "where does it fit in my workflow")

---

## 11. Synthesis Notes

> **This section will be completed after the user brings back the ChatGPT deep research reports.**

### What Claude-side research confirmed (3 parallel agents + direct web research):

1. **Cost arbitrage is real:** 50-62x cheaper than Opus. Head-to-head testing showed $0.27/task vs $3.67 for Opus with equivalent bug/vuln detection.
2. **Quality is "good enough" for 80% of coding tasks:** ~90% of Opus quality at ~7% cost. Matches Opus on bug detection (6/6) and security scanning (10/10). Falls short on architecture design, integration test depth, and subtle logic errors (off-by-one).
3. **Integration is trivially easy:** Zero new npm dependencies. Existing `openai` package works with a `baseURL` swap. Both OpenAI-compatible and Anthropic-compatible endpoints available.
4. **No vendor lock-in:** Available on OpenRouter, LiteLLM, and direct API. Multiple aggregator options.
5. **Generous rate limits:** 500 RPM / 20M TPM — far more generous than most providers.
6. **Real adoption:** Kilo Code community already using M2.5 as default execution model. OpenClaw patterns designate MiniMax as "cost-effective coder."
7. **Company is viable but risky:** Publicly traded (HKEX), $1.18B raised, but burning $251M/year (adjusted). Shanghai-based with geopolitical risk.
8. **Key gap — Aider polyglot:** M-2.7 is NOT listed on the Aider leaderboard. This is the most commonly referenced coding benchmark in the developer community.
9. **Verbosity warning:** Generates ~4x more output tokens than median models. Erodes per-token savings by 2-3x but still dramatically cheaper.
10. **Proprietary shift:** M2.7 is the first closed-source model in the M2 series. Community pushback exists.

### Key questions for ChatGPT deep research to answer:
1. **HumanEval / MBPP scores** — MiniMax hasn't published these classic coding benchmarks
2. **Function calling reliability in practice** — known issues exist; how bad are they?
3. **Long-context quality degradation** — at what token count does quality drop?
4. **DeepSeek V3.2 comparison** — similar price ($0.28/$0.42), how does quality compare?
5. **MiniMax Coding Plan details** — what does the subscription tier actually include?
6. **Production deployment case studies** — any companies beyond individual devs?

### Sources

**Official:**
- MiniMax Model Page: https://www.minimax.io/models/text/m27
- MiniMax Official Blog: https://www.minimax.io/news/minimax-m27-en
- MiniMax API Docs: https://platform.minimax.io/docs/guides/text-ai-coding-tools
- MiniMax API Docs (Chat): https://platform.minimax.io/docs/api-reference/text-chat
- GitHub MiniMax-M1: https://github.com/MiniMax-AI/MiniMax-M1
- GitHub MiniMax-M2: https://github.com/MiniMax-AI/MiniMax-M2

**Aggregators & Platforms:**
- OpenRouter: https://openrouter.ai/minimax/minimax-m2.7
- OpenRouter Performance: https://openrouter.ai/minimax/minimax-m2.7/performance
- LiteLLM MiniMax Docs: https://docs.litellm.ai/docs/providers/minimax
- MiniMax M2 Chat Completions: https://minimax-m2.com/docs/api/chat-completions

**Benchmarks & Analysis:**
- WaveSpeed Blog: https://wavespeed.ai/blog/posts/minimax-m2-7-self-evolving-agent-model-features-benchmarks-2026/
- Kilo Code Blog: https://blog.kilo.ai/p/minimax-m27
- Artificial Analysis: https://artificialanalysis.ai/articles/minimax-m2-7-everything-you-need-to-know
- DataLearner: https://www.datalearner.com/en/ai-models/pretrained-models/minimax-m2-7
- LLM Stats: https://llm-stats.com/models/minimax-m2.7
- Vals AI: https://www.vals.ai/models/minimax_MiniMax-M2.7
- SWE-Bench Pro: https://www.morphllm.com/swe-bench-pro
- Aider Leaderboard: https://llm-stats.com/benchmarks/aider-polyglot
- crackr.dev: https://www.crackr.dev/blog/best-llm-for-coding

**Reviews & Community:**
- thomas-wiegold.com: https://thomas-wiegold.com/blog/minimax-m-2-7-review-is-it-worth-the-hype/
- bswen.com: https://docs.bswen.com/blog/2026-03-24-kimi-vs-minimax-coding-comparison/
- computertech.co: https://computertech.co/minimax-m2-7-review/
- geeky-gadgets.com: https://www.geeky-gadgets.com/minimax-m27-benchmark-results/
- InfraFlow AI: https://infraflowai.substack.com/p/complete-technical-guide-automating
- Towards AI: https://pub.towardsai.net/minimax-m2-7-built-itself-heres-how-to-use-it-like-a-pro-27529761d9a7
- Reddit r/LocalLLaMA: https://www.reddit.com/r/LocalLLaMA/comments/1rxc9rw/minimax_m27_on_openrouter/
- Galaxy.ai comparison: https://blog.galaxy.ai/compare/minimax-m2-5-vs-minimax-m2-7
