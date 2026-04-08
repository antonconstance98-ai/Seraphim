# MiniMax M2.7 Deep Technical and Developer Economics Report

## Scope and evidence quality

This report focuses on what is publicly documentable about MiniMax M2.7 as of **Apr 3, 2026 (America/Chicago)**, and compares it (where possible) to Claude Opus 4.6, GPT-5.4, and DeepSeek V3 across the benchmarks you named (HumanEval, SWE-bench, MBPP, LiveCodeBench, Aider polyglot), plus newer agentic/SWE benchmarks that MiniMax and independent evaluators emphasize in 2025–2026.

A recurring theme is **benchmark mismatch**: many 2025–2026 “coding” claims are now reported on **repo-level / agentic** evaluations (SWE-* variants, terminal/tool benchmarks, agent task suites) rather than the older “single-function synthesis” tasks like HumanEval/MBPP. Even the authors of LiveCodeBench highlight that models can overfit HumanEval-style tasks and that performance can diverge on contamination-resistant “live” coding sets. citeturn12view0

Where MiniMax or third parties do not publish a number for M2.7 on a specific benchmark, this report **does not fabricate** one; it calls out the gap and, when helpful, provides competitor numbers from reputable leaderboards while documenting the absence for M2.7.

## Technical profile of MiniMax M2.7

### Architecture, parameter count, and active parameters

MiniMax’s most explicit public architecture disclosure (in English) is in its M2.1 post-training write-up: M2.1 “adopts a Mixture-of-Experts (MoE) architecture” with **~230B total parameters** and **~10B activated parameters**. citeturn17view0

For **M2.7 specifically**, MiniMax’s M2.7 launch materials and its public API docs/model pages emphasize capability, context, speed, and pricing, but (in the materials reviewed here) do **not** re-state the 230B/10B figures on the M2.7 model page or in the M2.7 API model card. citeturn2view1turn2view2turn15search1  
A number of third-party summaries assert that M2.7 is also a **230B MoE with ~10B active**; treat these as *unverified repeats* unless MiniMax publishes an M2.7-specific model card with those details. citeturn15search20turn0search35

**Best-supported conclusion:** MiniMax has publicly characterized the **M2-series** as large-scale MoE with low activated parameters; it is *plausible* (but not explicitly confirmed in the sources above) that M2.7 continues the same architectural family. citeturn17view0turn15search1

### Context window and maximum output

MiniMax M2.7’s model page lists:

- **Context window:** **204,800 tokens**  
- **Max output:** **128,000 tokens** (and **122,000** for the high-speed variant) citeturn2view1

These numbers are consistent with MiniMax’s positioning of M2.7 as a long-context model for “complex tasks” and coding/agent workflows. citeturn2view1turn15search1

### Throughput and tokens-per-second

MiniMax’s M2.7 model page lists a reference “typical token speed” of **~60 tokens/sec** (and **~100 tokens/sec** for “M2.7-highspeed”). citeturn2view1  
MiniMax’s API FAQ explains the TPS calculation (measuring from first to last output token) and notes TPS can fluctuate in practice; values shown on model pages are “reference values.” citeturn37view2

### Pricing model and caching primitives

MiniMax provides two primary billing modes for developers:

- **Pay-as-you-go (token metered)**
- **Token Plan (subscription with request quotas)** citeturn36view0turn37view1

For pay-as-you-go, MiniMax’s pricing page lists (text models):

- **MiniMax-M2.7:** **$0.30 / 1M input tokens**, **$1.20 / 1M output tokens**
- **MiniMax-M2.7-highspeed:** **$0.60 / 1M input**, **$2.40 / 1M output**
- **Prompt caching:** **$0.06 / 1M cache reads**, **$0.375 / 1M cache writes** (M2.7) citeturn38view0

This is an unusually low price point for a “frontier-ish” model, and it heavily influences any routing/cost-optimization strategy.

For Token Plan subscriptions, MiniMax sets **request quotas per 5 hours** (not token quotas) and offers monthly plans starting at **$10/month** (Starter) and **$20/month** (Plus). citeturn38view1turn37view1

## Benchmark performance in 2025–2026

image_group{"layout":"carousel","aspect_ratio":"16:9","query":["MiniMax M2.7 benchmark overview SWE Pro Multi-SWE VIBE-Pro GDPval-AA","MiniMax M2.7 SWE-Pro benchmark chart"],"num_per_query":1}

### What MiniMax officially emphasizes for M2.7

MiniMax’s M2.7 release includes a published “benchmark overview” graphic. citeturn6view0turn15search1  
In the same release narrative, MiniMax frames M2.7 as “deeply participating in its own evolution” and targets **real-world software engineering** and “agent teams” rather than classic single-function coding benchmarks. citeturn15search1turn2view1

From third-party coverage that repeats the release chart figures, M2.7 is associated with results including **SWE-Pro ~56.2%** and **GDPval-AA Elo ~1495**, plus TerminalBench values in the same family of evaluations. citeturn15search13turn41view0  
Because the detailed table is in an image, and different sites sometimes paraphrase it inconsistently, treat these numbers as “release-chart reported” rather than independently reproduced.

### Independent evaluations that are clearly published for M2.7

**Artificial Analysis (Mar 25, 2026)** characterizes M2.7 as improving over M2.5 on agentic tasks and hallucination behavior, and reports:

- **GDPval-AA Elo ~1494–1495** (improvement over M2.5) citeturn41view0
- Same per-token pricing as M2.5 ($0.30/$1.20) and a measured cost to run their “Intelligence Index” suite. citeturn41view0turn38view0
- “Availability: MiniMax first-party API only” (important for ecosystem/platform questions). citeturn41view0

These are not the classic benchmarks you listed, but they are among the most cited **independent 2025–2026** evaluations for modern coding agents.

### Coverage gaps on the classic coding benchmarks you requested

You asked specifically for **HumanEval, SWE-bench, MBPP, LiveCodeBench, Aider polyglot** comparisons versus Claude Opus 4.6, GPT-5.4, and DeepSeek V3.

Here is what is supported by the sources reviewed:

**HumanEval (classic)**
- A HumanEval leaderboard sourced from LayerLens includes Claude Opus 4.6, several OpenAI/DeepSeek entries, and pricing metadata, but it contains **no MiniMax M2.7 entry** (search within the page yields no match). citeturn18view0turn19view1turn19view0  
- Example competitor datapoint on that page: DeepSeek V3 is shown at **87.2** (pricing shown there is a separate dimension and may vary by reseller). citeturn18view0  
**Conclusion:** no well-cited, widely hosted HumanEval score for M2.7 was found in the reviewed sources.

**MBPP / MBPP+**
- The EvalPlus project is the modern home for MBPP+ and HumanEval+ style rigorous unit-test expansions, but the EvalPlus leaderboard page is not easily machine-readable in this environment and (more importantly) no public evidence in the reviewed materials indicates an EvalPlus MBPP(+)/HumanEval(+) run that includes M2.7. citeturn16view0turn15search7  
**Conclusion:** MBPP/MBPP+ for M2.7 is not documented in the sources assembled here.

**LiveCodeBench**
- A LiveCodeBench “compare scores” leaderboard (sourced from Artificial Analysis) includes many frontier models and includes MiniMax entries for **M2** and **M2.1**, but **does not list M2.7** (searching within the page finds no “M2.7”). citeturn20view0turn21view1turn21view0  
**Conclusion:** no published LiveCodeBench score for M2.7 was found in this dataset.

**Aider polyglot**
- Aider’s own polyglot leaderboard is publicly viewable and well-instrumented (showing cost for the benchmark run, malformed response rates, etc.), but **M2.7 is not present** in the currently captured portion of the leaderboard listings, and the page’s prominent entries focus heavily on OpenAI/Gemini/o-series runs from mid-2025. citeturn42view0  
**Conclusion:** as of the captured data, Aider polyglot does not offer a straightforward M2.7 comparison.

**SWE-bench**
- SWE-bench remains the most important “real repo issue fixing” benchmark in your list, but there are two practical complications:
  1. SWE-bench’s **public web UI** is interactive and data-backed by a large JSON file that was not retrievable in this environment, blocking direct extraction of a clean table of 2026 model scores.
  2. MiniMax’s own public M2.7 reporting emphasizes **SWE-Pro** and **Multi-SWE** style benchmarks more than SWE-bench Verified. citeturn33view0turn15search1turn6view0  

MiniMax’s **M2.5** release (Feb 2026) did claim **80.2% on SWE-bench Verified** and **51.3% on Multi-SWE-bench**, and also reports “matching” Opus 4.6 speed for that evaluation run. citeturn9search5  
However, that is **M2.5**, not M2.7. In contrast, a third-party guide claims an M2.7 “SWE-bench 78%” figure, but this is not corroborated by an official MiniMax benchmark table in the accessible docs. citeturn15search20

**Bottom line on benchmark comparison:**  
As of Apr 3, 2026, the strongest documented comparisons for M2.7 versus Claude Opus 4.6 and GPT-5.4 are on **agentic / workflow benchmarks** (GDPval-AA, SWE-Pro-like suites, tool/terminal tasks) via MiniMax’s release chart and third-party evaluation work, not on HumanEval/MBPP/LiveCodeBench/Aider polyglot. citeturn6view0turn41view0turn15search1

## Cost optimization for a developer spending $15/day on AI coding

### Price anchors you can rely on

**MiniMax M2.7 pay-as-you-go**: $0.30/M input, $1.20/M output; M2.7-highspeed is double that; prompt caching read/write pricing is explicitly listed. citeturn38view0

**Claude Opus 4.6 (Anthropic)**: Anthropic reports Opus 4.6 pricing starting at **$5/M input** and **$25/M output**, with “up to 90%” savings via prompt caching and “50%” via batch processing. citeturn40search0turn40search4turn40search1  
Anthropic also notes Opus 4.6 supports **1M token context (beta)** and **128k output tokens**, and introduces premium pricing for prompts exceeding **200k tokens**. citeturn40search1turn40search0

**GPT-5.4 (OpenAI API)**: OpenAI’s pricing table lists different rates by context tier and service class; for common “short context” usage, **gpt-5.4** is shown at **$1.25/M input** and **$7.50/M output** (with separate cached-input pricing), and higher for “long context.” citeturn40search2

**GPT-5.4 via ChatGPT Plus**: you stated you access GPT-5.4 via Plus ($20/mo). Because Plus is subscription-based and can have message/cap limits rather than linear token billing, it typically isn’t directly comparable to metered APIs for cost routing; routing decisions usually focus on what is **currently driving your variable spend** (your $15/day API usage).

### Context window comparison for cost planning

- M2.7: **204.8k context**. citeturn2view1  
- Opus 4.6: **1M context available in beta** on Anthropic’s platform; premium pricing beyond 200k tokens is explicitly called out. citeturn40search1turn40search0  
- GPT-5.4: OpenRouter lists **~1.05M context** for gpt-5.4, and OpenAI documents multiple context tiers/pricing; ChatGPT help docs also show tiered context windows in ChatGPT. citeturn40search12turn40search2turn40search3turn40search9  

**Practical implication:** M2.7’s context is *shorter than the largest* Opus 4.6/GPT-5.4 configurations, but it is already “big enough” to fit many mid-sized repos, multi-file patches, or large diff contexts—while remaining extremely cheap on input tokens. citeturn2view1turn38view0turn40search1

### How much could you save by routing tasks to M2.7?

Because you gave a daily spend ($15/day) rather than token counts, the most robust way to express savings is:

1) identify what fraction of your **token volume** can move to M2.7, and  
2) apply the **effective price ratio**, adjusted for retries and quality differences.

**Token-price ratio (Opus 4.6 vs M2.7):**  
Using Anthropic’s published Opus 4.6 price ($5/$25) and MiniMax’s M2.7 ($0.30/$1.20), M2.7 is typically about **~17–19× cheaper per token** for a coding workload depending on your output:input ratio. citeturn40search0turn38view0

If you reroute a fraction **f** of your daily API token volume from Opus 4.6 to M2.7, a first-order estimate is:

- **New daily cost ≈ $15 × [(1 − f) + f / 18]**

Examples (assuming similar retries/quality for the routed slice):
- Route **50%** of token volume → new cost ≈ **$7.92/day** (save ≈ **$7.08/day**) citeturn40search0turn38view0  
- Route **70%** → new cost ≈ **$5.08/day** (save ≈ **$9.92/day**) citeturn40search0turn38view0  
- Route **85%** → new cost ≈ **$2.96/day** (save ≈ **$12.04/day**) citeturn40search0turn38view0  

#### Quality-adjusted cost (retry-rate model)

If M2.7 requires more attempts on some tasks, adjust the routed slice by a retry multiplier **r**:

- **New daily cost ≈ $15 × [(1 − f) + (f × r) / 18]**

Even if M2.7 needs **20% more attempts** on the routed work (r = 1.2), the savings remain large at typical routing fractions because the base price gap is so wide. citeturn40search0turn38view0turn41view0  

A practical, developer-centric way to choose **where retries matter** is to route by task type:

### Recommended routing map by task type

M2.7 is most likely to be a large win when tasks are **high-volume** and **mechanically checkable** (compile/tests/lints), and when you benefit from its long context but don’t need the absolute best “agentic” success rates.

Use M2.7 first for:
- **Repo ingestion + summarization + retrieval scaffolding**: huge input tokens, moderate output; ideal for $0.30/M input. citeturn38view0turn2view1turn41view0  
- **Mechanical refactors** (rename/move, formatting, API migrations) verified by CI.  
- **Test generation, linters, type fixes** where failures give tight feedback loops.  
- **Bulk code writing** (boilerplate, interfaces, adapters) where you can run tests and iterate cheaply.

Keep Opus 4.6 / GPT-5.4 for:
- **Ambiguous, high-stakes changes** (security-sensitive auth, monetary logic, production incidents).  
- **Deep architectural decisions** where one-shot correctness saves time more than tokens.  
- **Very large-context reasoning** beyond ~200k when you can’t compact context safely (Opus 4.6 1M beta / GPT-5.4 large-context modes). citeturn40search1turn40search2turn40search9  

### Free tier and credits

MiniMax does not present a “free forever” developer tier in the surfaced docs, but it does offer:

- A **low-cost Token Plan** starting at **$10/month** (Starter) and **$20/month** (Plus), with request quotas that reset on a rolling 5-hour window. citeturn38view1turn37view1  
- A Token Plan **referral program** is listed in the docs navigation (details not expanded here). citeturn37view1  

For startup credits beyond these plans (e.g., dedicated startup grants), the docs link out to a “Developer Program” document, but it is hosted externally and was not fully reviewable in this environment. citeturn37view0turn37view1

## MiniMax M2.7 API for developers

### Endpoints and compatibility layers

MiniMax documents **two major compatibility surfaces**:

**Anthropic-compatible endpoint**
- International: `https://api.minimax.io/anthropic` (Claude Code config guidance) citeturn36view0  
- China: `https://api.minimaxi.com/anthropic` citeturn36view0  
MiniMax’s OpenCode config example uses `https://api.minimax.io/anthropic/v1` as the Anthropic-style base URL. citeturn36view0  

**OpenAI-compatible endpoint**
- International: `https://api.minimax.io/v1` citeturn36view0  
- China: `https://api.minimaxi.com/v1` citeturn36view0  

This “dual-stack compatibility” is one reason MiniMax can plug into many coding tools that already support OpenAI- or Anthropic-shaped APIs. citeturn36view0turn38view2

### Authentication and API key setup

MiniMax’s API FAQ instructs developers to create/manage keys via **Account → Settings → API Keys**, and emphasizes that keys should not be exposed client-side and may be disabled if leaked. citeturn37view2

MiniMax also distinguishes between:
- **Token Plan API keys** (subscription/quota-based)  
- **Pay-as-you-go API keys** (token billed)  
and notes the key type determines billing behavior. citeturn36view0turn37view1

### SDK support and Node/Python packages

MiniMax’s “Integrate via SDK” quickstart explicitly recommends using the **Anthropic SDK** (Python / Node.js) against MiniMax’s Anthropic-compatible surface rather than an official MiniMax-native SDK:

- Python install: `pip install anthropic`  
- Example call uses `client.messages.create(model="MiniMax-M2.7", ...)` citeturn38view2  

For Node/open-source tool integration, MiniMax’s OpenCode configuration shows use of `@ai-sdk/anthropic` with a MiniMax baseURL, indicating compatibility with Vercel’s AI SDK ecosystem (via an Anthropic provider). citeturn36view0  

**What’s missing:** In the reviewed MiniMax docs, there is no clearly documented “official minimax” Python or Node SDK package; the first-party guidance is to use Anthropic/OpenAI-compatible libraries. citeturn38view2turn36view0

### Pricing summary for M2.7 (exact)

From MiniMax’s pay-as-you-go pricing table:

- **MiniMax-M2.7:** $0.30/M input, $1.20/M output  
- **MiniMax-M2.7-highspeed:** $0.60/M input, $2.40/M output  
- **Prompt caching:** $0.06/M cache read; $0.375/M cache write citeturn38view0

For comparison points you requested:
- **Claude Sonnet tier (reported):** commonly stated as ~$3/M input and ~$15/M output for modern Sonnet-class models; one summary explicitly states Sonnet 4.6 at $3/$15 and Opus 4.6 at $5/$25. Treat this as secondary unless you validate against Anthropic’s pricing page directly. citeturn40search8turn40search4  
- **Claude Opus 4.6 (Anthropic):** $5/M input, $25/M output (plus caching/batch options). citeturn40search0turn40search4  
- **GPT-5.4 (OpenAI API):** $1.25/M input & $7.50/M output (short context), higher for long-context tiers; cached-input pricing is also listed. citeturn40search2  

### Rate limits and throttling

MiniMax exposes rate limiting in two main ways:

- **Token Plan:** explicit **requests per 5 hours** (e.g., Starter 1,500 req/5h; Plus 4,500; Max 15,000). citeturn38view1turn37view1  
- **Pay-as-you-go:** MiniMax references a rate limit help page from the “AI coding tools” guide, but the exact per-minute/per-day throttles are not stated in the captured excerpt. citeturn36view0  

### Regional availability and latency considerations

MiniMax explicitly provides different base domains depending on whether you are an international user or in China (`api.minimax.io` vs `api.minimaxi.com`), which is typically done to address network routing/latency and regulatory constraints. MiniMax does not publish a formal latency SLA in the captured docs, but the endpoint bifurcation is clearly documented. citeturn36view0

## Platform availability and ecosystem coverage

### Is M2.7 on OpenRouter, Together, Fireworks, LiteLLM, Azure, or AWS?

The most direct independent statement found is from Artificial Analysis: **“Availability: MiniMax first-party API only.”** citeturn41view0turn15search11

That implies:
- **OpenRouter / Together AI / Fireworks AI (as hosted resellers):** not officially listed as primary distribution channels for M2.7 in the sources above.  
- **Azure / AWS managed model catalogs:** no evidence in the reviewed sources that M2.7 is distributed via those clouds’ managed foundation model services.

However, “first-party API only” does *not* mean you can’t use M2.7 in common tooling:
- MiniMax documents direct configuration for many popular coding tools and agent frameworks, including Anthropic-compatible wiring for Claude Code and OpenAI-compatible wiring for tools that accept OpenAI endpoints. citeturn36view0  
- M2.7 is also presented through at least one third-party developer tool wrapper (for example, an Ollama “cloud” entry), which appears to act as a client-side integration rather than an official hosted reseller. citeturn15search17  

### Practical latency and deployment advice

If you are in the US (America/Chicago), you would typically use `api.minimax.io` endpoints (per MiniMax’s “international users” guidance). citeturn36view0  
For latency-sensitive agent loops, MiniMax’s **highspeed** variant is explicitly priced and exposed, and the model page reports a higher reference output TPS for that tier. citeturn2view1turn38view0

## Code examples and primary source URLs

### Anthropic-compatible usage (Python)

```python
# Based on MiniMax docs: use the Anthropic SDK to call MiniMax-M2.7
# Install: pip install anthropic

import anthropic

client = anthropic.Anthropic()

message = client.messages.create(
    model="MiniMax-M2.7",
    max_tokens=1000,
    system="You are a helpful assistant.",
    messages=[
        {"role": "user", "content": [{"type": "text", "text": "Hi, how are you?"}]}
    ],
)

for block in message.content:
    if block.type == "text":
        print(block.text)
```

### Claude Code (Anthropic-compatible) wiring

```json
// ~/.claude/settings.json (MiniMax docs excerpt)
// International endpoint:
{
  "env": {
    "ANTHROPIC_BASE_URL": "https://api.minimax.io/anthropic",
    "ANTHROPIC_AUTH_TOKEN": "<MINIMAX_API_KEY>",
    "ANTHROPIC_MODEL": "MiniMax-M2.7"
  }
}
```

### OpenAI-compatible base URL (example: Codex CLI profile)

```toml
# MiniMax docs excerpt: configure an OpenAI-compatible base URL for MiniMax
[model_providers.minimax]
name = "MiniMax Chat Completions API"
base_url = "https://api.minimax.io/v1"
env_key = "MINIMAX_API_KEY"
wire_api = "chat"
requires_openai_auth = false

[profiles.m27]
model = "codex-MiniMax-M2.7"
model_provider = "minimax"
```

### Primary sources (URLs)

```text
MiniMax M2.7 model page (context, max output, TPS):
https://www.minimax.io/models/text/m27

MiniMax M2.7 launch post (includes benchmark overview graphic):
https://www.minimax.io/news/minimax-m27-en

MiniMax API docs (M2.7 model card + context + TPS):
https://platform.minimax.io/docs/models/minimax-m2.7

MiniMax pricing (pay-as-you-go token pricing + caching):
https://platform.minimax.io/docs/guides/pricing-paygo

MiniMax Token Plan overview and pricing (request quotas / 5-hour rolling window):
https://platform.minimax.io/docs/token-plan/intro
https://platform.minimax.io/docs/guides/pricing-token-plan

MiniMax AI coding tools guide (OpenAI-compatible + Anthropic-compatible endpoints):
https://platform.minimax.io/docs/guides/text-ai-coding-tools

Anthropic Opus 4.6 pricing/context notes:
https://www.anthropic.com/claude/opus
https://www.anthropic.com/news/claude-opus-4-6
https://platform.claude.com/docs/en/about-claude/pricing

OpenAI API pricing (gpt-5.4):
https://developers.openai.com/api/docs/pricing/

OpenAI help (GPT-5.4 context notes in ChatGPT tiers):
https://help.openai.com/en/articles/11909943-gpt-53-and-gpt-54-in-chatgpt

Artificial Analysis (independent evaluation summary for M2.7):
https://artificialanalysis.ai/articles/minimax-m2-7-everything-you-need-to-know

LiveCodeBench overview (benchmark methodology context):
https://livecodebench.github.io/

Aider polyglot leaderboards:
https://aider.chat/docs/leaderboards/
```

