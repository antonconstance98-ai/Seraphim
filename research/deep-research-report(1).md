# MiniMax M-2.7 Developer Ecosystem in 2026

## Executive summary

MiniMax M‑2.7 (often shown as **MiniMax‑M2.7**) is positioned in 2026 as a long‑context (204,800 tokens) “agentic” coding model available via **MiniMax’s first‑party API**, with two major “ecosystem bridge” surfaces: **OpenAI‑compatible** (`/v1`) and **Anthropic‑compatible** (`/anthropic`) endpoints. citeturn11view0turn12view0turn10view0

From a developer ecosystem perspective, MiniMax’s own documentation explicitly targets mainstream “AI coding tool” workflows (Claude Code, Cursor, VS Code extensions like Cline/Roo Code/Kilo Code, plus CLI tools like OpenCode and Grok CLI), and emphasizes that M2.7’s best results depend on **preserving full tool-call / reasoning artifacts across turns** (“interleaved thinking”), not just the final text. citeturn27view0turn11view0turn34view0

Interoperability is unusually strong in 2026 because MiniMax supplies **official MCP servers** (Python + JavaScript) for multimodal tools, plus a **Token Plan “Coding Plan MCP”** toolchain that adds `web_search` and `understand_image` to coding workflows through MCP clients like Claude Code and Cursor. citeturn7view2turn34view1turn33search0turn33search6

In the broader routing/gateway ecosystem, M2.7 is distributed via **OpenRouter** under `minimax/minimax-m2.7` (release date shown as March 18, 2026; pricing shown as $0.30/M input and $1.20/M output; 204,800 context), and can be consumed with OpenAI‑compatible code by swapping base URL + API key. citeturn25search2turn25search0turn25search9turn25search17

Community feedback in early 2026 is polarized but actionable: developers praise the value/throughput and “agent loop” behavior (reading more context before writing), while also reporting friction around **tool dropdown model registries**, **context window metadata mismatches**, **text‑only modality limits**, and uncertainty about **open‑weights licensing/timing** (with third‑party analysis stating no open‑weights announcement yet, while a MiniMax org reply on Hugging Face suggests open‑sourcing “in 2 weeks” and that M2.7’s parameter size matches M2.5). citeturn24view0turn21search22turn26search26turn23search3turn8view0

## What M‑2.7 looks like to developers

MiniMax markets M2.7 as a coding/agent model with a fast variant (“M2.7‑highspeed”), and provides multiple API styles so developers can reuse existing OpenAI/Anthropic tooling with minimal changes. citeturn10view0turn11view0turn12view0

### Model IDs, context, and speed tiers

Under the OpenAI‑compatible docs, MiniMax lists:

- `MiniMax-M2.7` — **204,800** context; output speed “~60 tps”  
- `MiniMax-M2.7-highspeed` — **204,800** context; output speed “~100 tps” citeturn11view0

OpenRouter’s model page for `minimax/minimax-m2.7` echoes the **204,800 context** and shows a “Released Mar 18, 2026” date. citeturn25search2turn7view1

### API surfaces: native, OpenAI‑compatible, Anthropic‑compatible

MiniMax’s ecosystem story depends on three complementary surfaces:

- **Native MiniMax endpoint** shown in marketing examples: `https://api.minimax.io/v1/text/chatcompletion_v2` citeturn10view0  
- **OpenAI‑compatible** base URL: `https://api.minimax.io/v1` citeturn11view0  
- **Anthropic‑compatible** base URL: `https://api.minimax.io/anthropic` citeturn12view0  

These compatibility interfaces are explicitly promoted as “meet developers’ needs for the OpenAI/Anthropic API ecosystem.” citeturn11view0turn12view0

### Tool calling + “interleaved thinking” is stateful, not optional

MiniMax’s guidance is consistent across their OpenAI and Anthropic compatibility docs: in multi‑turn tool/function calling, you must append the **complete model response** (including tool call fields and reasoning artifacts) back into the conversation history. citeturn11view0turn12view0turn27view0

A key operational detail: the OpenAI‑compatible interface notes that the assistant `content` may contain `<think>` tags that “must be preserved completely,” and that `reasoning_split=True` can emit a developer‑friendly `reasoning_details` field that also must be preserved. citeturn11view0

### Parameter limitations that matter in production

MiniMax documents several “gotcha” mismatches vs upstream OpenAI semantics:

- `temperature` range is constrained to **(0.0, 1.0]**, and out‑of‑range values error. citeturn11view0  
- Some OpenAI parameters (`presence_penalty`, `frequency_penalty`, `logit_bias`, etc.) may be **ignored**. citeturn11view0  
- `n` supports only **1** on the OpenAI‑compatible interface. citeturn11view0  
- Deprecated `function_call` is not supported; use `tools`. citeturn11view0  

On the Anthropic‑compatible interface, MiniMax lists “ignored” parameters including `mcp_servers` and `context_management`, and states image/document inputs are not currently supported there. citeturn12view0

## Developer tooling and framework support matrix

MiniMax’s own “AI coding tools” guide is the closest thing to an official ecosystem map: it enumerates supported clients and provides concrete configuration steps for each. citeturn34view0turn3view0

### Comparison table of tools and frameworks

| Tool | Type | Support level | Notes | URL |
|---|---|---|---|---|
| **OpenAI‑compatible MiniMax API** | API surface | Native | `OPENAI_BASE_URL=https://api.minimax.io/v1` and `model="MiniMax-M2.7"`; preserve `<think>`/`reasoning_details`; some params ignored; `n=1`; `temperature∈(0,1]` | citeturn11view0 |
| **Anthropic‑compatible MiniMax API** | API surface | Native | `ANTHROPIC_BASE_URL=https://api.minimax.io/anthropic`; supports text + tool use + thinking blocks; no image/document input; several Anthropic params ignored (incl. `mcp_servers`) | citeturn12view0 |
| **miniMax native `chatcompletion_v2`** | API endpoint | Native | Example endpoint `https://api.minimax.io/v1/text/chatcompletion_v2` used in official marketing quickstart | citeturn10view0 |
| **Prompt caching (MiniMax)** | Platform feature | Native | Automatic caching support documented; best practice: keep repeated/system content at start; track cache tokens in usage | citeturn30view0turn31view0 |
| **MiniMax MCP server** | MCP server | Native (official) | Official MCP server for TTS, image/video generation, etc.; advertised as compatible with Cursor / Claude Desktop / Windsurf / OpenAI Agents | citeturn7view2turn33search6 |
| **MiniMax Coding Plan MCP** | MCP server | Native (official) | Provides `web_search` + `understand_image`; install via `uvx minimax-coding-plan-mcp`; PyPI version shown as **0.0.4** | citeturn34view1turn33search0 |
| **Claude Code** | CLI coding agent | Officially documented (adapter) | MiniMax docs recommend it; requires pointing Claude Code to MiniMax Anthropic‑compatible endpoint + API key | citeturn34view0turn3view0 |
| **Cursor** | IDE/editor | Officially documented (adapter) | MiniMax docs include “Use in Cursor” for Token Plan MCP (`mcp.json` with `uvx minimax-coding-plan-mcp`) | citeturn34view1turn7view2 |
| **Cline (VS Code extension)** | IDE extension | Officially documented | MiniMax docs specify Cline must be **≥3.47.0**; select provider “MiniMax” and model “MiniMax‑M2.7” | citeturn3view0turn34view0 |
| **Roo Code** | IDE extension | Officially documented; some lag in registries | MiniMax docs include Roo Code setup; GitHub issue reports M2.7 missing in dropdown (app v3.51.1) | citeturn16search3turn21search22 |
| **Kilo Code** | IDE extension | Officially documented | MiniMax docs include VS Code install steps + selecting provider “MiniMax” and model “MiniMax‑M2.7” | citeturn16search3turn34view0 |
| **OpenCode** | CLI coding agent | Officially documented | Install via curl/npm; configure provider + model; supports MiniMax Anthropic endpoint config JSON example | citeturn16search3turn34view1 |
| **OpenClaw** | Agent platform | Officially documented | OpenClaw docs show provider config for MiniMax (Anthropic‑messages API) and `minimax/MiniMax-M2.7`; fix referenced in OpenClaw release 2026.1.12 | citeturn25search21turn29view0 |
| **LiteLLM** | Gateway/proxy SDK | Via provider + proxy (adapter) | MiniMax provider docs show routing to MiniMax `/v1` and `/anthropic/v1/messages`; examples use M2.1 but same pattern applies; recent issues track adding OpenRouter M2.7 pricing and Ollama Cloud streaming bugs | citeturn7view0turn26search6turn26search22 |
| **OpenRouter** | Router/aggregator | Via router (adapter) | `minimax/minimax-m2.7` listed with 204,800 context and $0.30/$1.20 pricing; OpenAI‑compatible base URL `https://openrouter.ai/api/v1`; tool calling guide available | citeturn25search2turn25search17turn25search3 |
| **LangChain** | Agent framework | Via provider integration + OpenAI‑compatible | LangChain docs include a “Minimax integrations” page (historically includes `MINIMAX_API_KEY` + group id); separate LangChain OpenRouter integration exists | citeturn6search12turn25search27 |
| **LlamaIndex** | Agent/RAG framework | Native provider module | LlamaIndex docs specify `pip install llama-index-llms-minimax` and show `MiniMax(model="MiniMax-M2.7")`; release notes indicate provider integration landing as `llama-index-llms-minimax [0.1.0]` | citeturn6search1turn6search5 |
| **Microsoft AutoGen** | Agent framework | Via OpenAI‑compatible adapter | AutoGen’s OpenAI client supports OpenAI‑compatible endpoints “not tested or guaranteed” for non‑OpenAI models; works by swapping base URL + key | citeturn6search14turn6search4 |

### Key setup snippets (canonical patterns)

MiniMax’s docs converge on a few “canonical” integration patterns.

**OpenAI SDK → MiniMax (OpenAI‑compatible format)** citeturn11view0
```bash
export OPENAI_BASE_URL="https://api.minimax.io/v1"
export OPENAI_API_KEY="YOUR_MINIMAX_KEY"
```

**Anthropic SDK → MiniMax (Anthropic‑compatible format)** citeturn12view0
```bash
export ANTHROPIC_BASE_URL="https://api.minimax.io/anthropic"
export ANTHROPIC_API_KEY="YOUR_MINIMAX_KEY"
```

**Tool calling continuity rule (core “gotcha”)**: preserve the *full* assistant response in history (tool calls + reasoning), not just rendered text; MiniMax repeats this requirement across OpenAI‑compatible and Anthropic‑compatible interfaces and in its “Tool Use & Interleaved Thinking” guide. citeturn11view0turn12view0turn27view0

## Compatibility with LiteLLM, OpenRouter, and MCP servers

### LiteLLM

**Availability**: LiteLLM documents a MiniMax provider with both (a) OpenAI‑style `/v1` routing and (b) Anthropic Messages routing using `https://api.minimax.io/anthropic/v1/messages` in examples. citeturn7view0turn32view2

**Setup (direct calls)**: LiteLLM’s docs show the pattern of passing `api_key` plus `api_base` to route to MiniMax. citeturn7view0turn32view2

**Setup (LiteLLM proxy as “adapter layer”)**: LiteLLM documents proxy YAML configuration to expose MiniMax models behind a local OpenAI‑style or Anthropic‑style endpoint, letting existing SDKs talk to the proxy instead of MiniMax directly. citeturn7view0

**Known limitations / operational risks** (as observed in the 2026 ecosystem):

- Model registry/pricing metadata can lag new releases: a LiteLLM issue explicitly tracks adding `openrouter/minimax/minimax-m2.7` pricing/context metadata. citeturn26search6  
- Adapter edge cases exist: a LiteLLM issue reports `minimax-m2.7:cloud` (Ollama Cloud) failing on subsequent requests and suggests the bug is in the adapter’s streaming handling, not the underlying model. citeturn26search22  
- **Supply chain security** became a concrete risk in March 2026: LiteLLM published a security update about a suspected supply chain incident and announced a “clean” release (v1.83.0) built under a hardened pipeline. citeturn33search10turn33search17  

### OpenRouter

**Availability**: OpenRouter lists the model as `minimax/minimax-m2.7` with **204,800 context**, release date shown as **Mar 18, 2026**, and pricing shown as **$0.30/M input** and **$1.20/M output**. citeturn25search2turn7view1

**Setup**: OpenRouter’s docs describe an OpenAI‑compatible API schema and provide an OpenAI SDK integration example that sets `baseURL` to `https://openrouter.ai/api/v1`. citeturn25search9turn25search17

**Tool calling**: OpenRouter provides a standardized tool calling interface and documentation for tools/function calling across models. citeturn25search3turn25search12

**Known limitations (practical)**: Because OpenRouter is a router across multiple providers, capabilities that depend on MiniMax‑specific semantics (especially tool/reasoning preservation expectations) can be harder to debug unless you pin routing behavior; OpenRouter’s own best‑practices doc notes “sticky routing” to maximize caching hit rates across providers. citeturn28search25turn25search2

### MCP servers in the MiniMax ecosystem

MiniMax’s MCP story in 2026 has two layers: an **official multimodal MCP server** (general‑purpose) and a **Token Plan “Coding Plan MCP”** (coding‑workflow‑focused).

**Official MiniMax MCP server (multimodal)**

- MiniMax’s MCP guide states MiniMax provides official MCP server implementations (Python + JavaScript) and that these can be accessed from MCP clients such as Claude Desktop, Cursor, Windsurf, and OpenAI Agents. citeturn7view2turn33search6  
- The GitHub repository description emphasizes that this MCP server enables interaction with MiniMax TTS, image generation, and video generation APIs, and is MIT licensed. citeturn33search6  

**Token Plan “Web Search & Image Understanding MCP” (coding plan MCP)**

- MiniMax’s Token Plan MCP guide describes two tools—`web_search` and `understand_image`—as part of coding workflows, and shows installation with `uvx` plus integration steps for Claude Code and Cursor. citeturn34view1  
- The PyPI project page identifies `minimax-coding-plan-mcp` as a specialized MCP server for coding plan users and shows latest version **0.0.4** (released Feb 10, 2026). citeturn33search0  

**Key limitation that surprises teams**: the Anthropic‑compatible API explicitly marks `mcp_servers` as “ignored,” meaning “MCP integration” is not something you pass to the Anthropic request payload; it’s a **client/tooling configuration** (Claude Code/Cursor/etc.) or a separate server integration. citeturn12view0turn34view1

## Community feedback on using M‑2.7 for production code generation

This section summarizes representative community signal across Reddit, GitHub issues, and third‑party reviews as of early April 2026 (America/Chicago). citeturn25search2turn24view0turn13search18turn21search22

### High-level sentiment themes

Across posts and issues, the most common “production” themes cluster into five buckets:

- **Value-per-token**: many comparisons emphasize that M2.7 is priced far below frontier closed models while approaching them on coding/agent tasks. citeturn21search11turn25search26turn31view0turn24view0  
- **Agent-loop behavior**: users report M2.7 often “reads more” before editing, which can improve correctness but risks timeouts in agent harnesses. citeturn13search18turn27view0  
- **Tooling friction**: early integration breakage (model missing in dropdowns; wrong context window metadata; “unknown model” errors) is repeatedly reported. citeturn21search22turn26search26turn25search21turn21search30  
- **Modality reality vs expectations**: M2.7 is frequently described as “text‑only,” but developers use MCP to regain web search and “vision” through tools; confusion persists when users expect direct vision input via the Anthropic/OpenAI chat payloads. citeturn24view0turn12view0turn34view1turn21search18  
- **Licensing uncertainty**: third‑party analysts explicitly state MiniMax had not announced open‑weights for M2.7 as of Mar 25, 2026, while a MiniMax org reply on Hugging Face suggested open‑sourcing in “2 weeks” and confirmed M2.7’s parameter size matches M2.5. citeturn24view0turn23search3turn8view0  

### Representative quotes (with labels; URLs in appendix)

> **[Q1 | r/LocalLLaMA benchmark behavior]** “MiniMax‑M2.7 reads extensively before writing… On tasks where that extra context pays off, it catches things other models miss.” citeturn13search18

> **[Q2 | r/LocalLLaMA open-weights concern]** “Minimax 2.7 might not have the same performance when run locally… Could this be the reason blocking the release…?” citeturn21search0

> **[Q3 | r/ChatGPT alignment framing]** “It’s not some dumb censorship anymore, it’s some Claude‑level alignment.” citeturn21search1

> **[Q4 | GitHub issue: Roo Code dropdown]** “The MiniMax‑M2.7 model is missing from the dropdown menu… making it impossible to select.” citeturn21search22

> **[Q5 | GitHub issue: Cline context metadata]** “The context window is shown 192K… But according to official doc it is 204,800.” citeturn26search26

> **[Q6 | Third‑party analysis on licensing]** “MiniMax has not announced whether MiniMax‑M2.7 will be open weights.” citeturn24view0

### What “production” feedback implies for engineering teams

A consistent throughline is that teams succeed with M2.7 when they treat it as a **tool-using state machine** rather than a pure “code completion” model: keep prompts structured, preserve complete tool/reasoning state between turns, exploit caching, and accept that IDE integrations may lag the model release by days/weeks. citeturn27view0turn30view0turn21search22turn25search21

## Practical gotchas and mitigation strategies

### Gotcha: tool-call and reasoning continuity will silently degrade quality

**Symptom**: agent harnesses “feel worse than benchmarks,” especially on multi‑step tasks; tool use becomes inconsistent.

**Root cause**: MiniMax explicitly requires appending the full assistant response (including tool calls and reasoning artifacts) back into the conversation history for multi‑turn tool calling. citeturn11view0turn12view0turn27view0

**Mitigation** (OpenAI‑compatible): preserve `<think>` content or preserve `reasoning_details` if using `reasoning_split=True`. citeturn11view0turn27view0

**Mitigation** (Anthropic‑compatible): append the entire `response.content` list (thinking/text/tool_use) into history. citeturn12view0turn27view0

### Gotcha: API parameter mismatches vs OpenAI/Anthropic defaults

**Operational constraints** that can break production if you rely on upstream defaults:

- `temperature` must be in **(0.0, 1.0]** on MiniMax’s OpenAI‑compatible API. citeturn11view0  
- Some OpenAI parameters are ignored; `n=1`; legacy `function_call` not supported. citeturn11view0  
- Anthropic‑compatible ignores several fields including `mcp_servers` and does not support image/document message blocks. citeturn12view0  

**Mitigation**: enforce provider-specific validation and request normalization at the gateway layer (e.g., reject or rewrite unsupported parameters before sending). MiniMax’s docs make these constraints explicit enough to encode as checks in an SDK wrapper. citeturn11view0turn12view0

### Gotcha: text‑only model, “vision” comes via MCP tooling

Artificial Analysis describes M2.7 as “text input and output only (no multimodality).” citeturn24view0  
MiniMax’s Anthropic‑compatible interface similarly states image/document inputs are not supported. citeturn12view0

**Mitigation**: use MiniMax’s Token Plan MCP (`understand_image`) for image understanding workflows, configured in Claude Code/Cursor (or another MCP client), rather than expecting direct image input support in the chat payload. citeturn34view1turn12view0

### Gotcha: context-window “paper cuts” in IDE integrations

Even when the underlying model supports 204,800 tokens, IDE plugins can lag in UI metadata and model registries (e.g., context showing 192K; missing model in dropdown). citeturn11view0turn26search26turn21search22

**Mitigation**: prefer tools that allow **manual model ID entry** or raw provider config, and validate the effective context in real calls. OpenClaw’s docs, for example, show explicit provider configuration and also document an “Unknown model” error mode and its fix. citeturn25search21

### Gotcha: long tasks can terminate early near the limit

MiniMax’s own “M2.7 Usage Tips” warns that when using tools with context compression (example: Claude Code), M2.7 may terminate tasks early when approaching context capacity thresholds. citeturn29view0turn11view0

**Mitigation**: adopt a “multi‑window workflow” for long work (phase tasks, externalize tests/scripts, and restart when switching goal scopes), mirroring MiniMax’s recommended patterns. citeturn29view0turn27view0

### Cost + performance tradeoffs

MiniMax’s Pay‑As‑You‑Go pricing table shows:

- M2.7: $0.30/M input; $1.20/M output; cache read $0.06/M; cache write $0.375/M  
- M2.7‑highspeed: $0.60/M input; $2.40/M output (cache read/write same as M2.7) citeturn31view0turn30view1

MiniMax also markets automatic caching (no app changes required) and describes how to maximize cache utility by front‑loading stable prompt/tool content. citeturn30view0turn10view0

**Mitigation**: for agentic coding tools with large repeated system/tool scaffolds, caching strategy can dominate costs; implement “static prefix discipline” and evaluate cache hit rates via returned usage fields. citeturn30view0turn30view1turn31view0

### Local deployment (open weights) and quantization: what can be said rigorously

As of Mar 25, 2026, Artificial Analysis states MiniMax had not announced whether M2.7 will be open weights, and describes availability as “MiniMax first‑party API only.” citeturn24view0  
However, a MiniMax org reply on Hugging Face (in a discussion on the M2.5 model) says M2.7’s parameter size is the same as M2.5 and that they “will opensource in 2 weeks.” citeturn23search3  
Attempts to access the Hugging Face weights page linked from OpenRouter returned an authorization error (401), indicating gated access at minimum. citeturn8view0

**Inference (clearly labeled)**: if M2.5’s published weights are 230B total / 10B active MoE (as widely documented for the M2 series), and MiniMax states M2.7 has the same parameter size as M2.5, then M2.7 likely inherits similar “large‑weights, sparse‑active” deployment characteristics when/if open weights ship. This inference relies on the size equivalence statement plus M2‑series public specs, not a released M2.7 model card. citeturn23search3turn16search0turn17view0

For the M2 series that *is* openly deployable, vLLM’s recipes document very large memory requirements (e.g., “220 GB for weights”) and specialized reasoning/tool parsing, reinforcing that “local deployment” is typically multi‑GPU territory unless heavily quantized. citeturn17view0turn18search2turn18search13

## Notable projects and ecosystem usage signals

### MiniMax’s own “first-party” product surfaces

MiniMax states M2.7 is available on its own Agent product and API platform and publishes a detailed narrative of using internal M2.7 to accelerate reinforcement‑learning workflows. citeturn9view0turn13search31

### Open-source and developer toolchain integrations actively tracking M2.7

The speed at which open-source projects file issues/PRs for “add M2.7 model ID” is itself an ecosystem signal:

- **Roo Code** issue: model missing in provider dropdown; highlights how model registries lag releases. citeturn21search22  
- **OpenCode** issue: request to add M2.7 to provider list. citeturn21search30  
- **Spring AI** issue: request to add M2.7 model support. citeturn21search10  
- **OpenWebUI** issue: reports M2.7 “not working well” via OpenRouter configuration. citeturn26search2  

### Agent frameworks and configs “in the wild” naming MiniMax‑M2.7 explicitly

Several repositories (examples surfaced via search) include config stanzas calling MiniMax‑M2.7 with OpenAI‑compatible base URLs, indicating practical adoption patterns:

- A ByteDance repo (`deer-flow`) includes example YAML showing MiniMax‑M2.7 configured via `langchain_openai:ChatOpenAI` with `base_url: https://api.minimax.io/v1` and the documented temperature constraint. citeturn26search13turn11view0  
- A repo (“sirchmunk”) shows code using `base_url="https://api.minimax.io/v1"` and `model="MiniMax-M2.7"`. citeturn26search25  
- A Cursor-focused rules repo explicitly markets itself as “Built for MiniMax M2.7… aligned with the official release and API docs.” citeturn26search12  

### “Cloud local” distribution via Ollama Cloud

Ollama’s library page shows “minimax‑m2.7:cloud” and lists one‑command launch integrations for Claude Code, Codex, OpenCode, and OpenClaw, providing an alternate path for developers who want a single CLI launcher experience rather than direct API wiring. citeturn26search19

## Integration paths flowchart

```mermaid
flowchart TD
  A[M2.7 model access] --> B[MiniMax first-party API]
  A --> C[OpenRouter routing]
  A --> D[LiteLLM proxy/gateway]

  B --> B1[OpenAI-compatible /v1]
  B --> B2[Anthropic-compatible /anthropic]
  B --> B3[Native chatcompletion_v2]

  B2 --> E[AI coding tools via Anthropic-style clients]
  B1 --> F[Agent frameworks via OpenAI-style clients]

  E --> E1[Claude Code]
  E --> E2[Cursor]
  E --> E3[OpenCode]
  E --> E4[VS Code extensions (Cline/Roo/Kilo)]

  F --> F1[LlamaIndex]
  F --> F2[LangChain]
  F --> F3[AutoGen]

  B --> G[MCP servers]
  G --> G1[Official MiniMax MCP (multimodal tools)]
  G --> G2[Coding Plan MCP: web_search + understand_image]

  E1 --> G2
  E2 --> G2
  E3 --> G2
```

This diagram reflects MiniMax’s documented “AI coding tools” integrations plus their MCP guides and OpenRouter/LiteLLM ecosystem patterns. citeturn34view0turn34view1turn7view2turn25search2turn7view0

## Source URLs

```text
OFFICIAL / PRIMARY (MiniMax)
- https://www.minimax.io/news/minimax-m27-en
- https://www.minimax.io/models/text/m27
- https://platform.minimax.io/docs/api-reference/text-openai-api
- https://platform.minimax.io/docs/api-reference/text-anthropic-api
- https://platform.minimax.io/docs/guides/text-m2-function-call
- https://platform.minimax.io/docs/guides/text-ai-coding-tools
- https://platform.minimax.io/docs/token-plan/best-practices
- https://platform.minimax.io/docs/api-reference/text-prompt-caching
- https://platform.minimax.io/docs/api-reference/anthropic-api-compatible-cache
- https://platform.minimax.io/docs/guides/pricing-paygo
- https://platform.minimax.io/docs/guides/mcp-guide
- https://platform.minimax.io/docs/token-plan/mcp-guide

OFFICIAL / PRIMARY (MCP repos)
- https://github.com/minimax-ai/minimax-mcp
- https://pypi.org/project/minimax-coding-plan-mcp/
- https://github.com/MiniMax-AI/MiniMax-Coding-Plan-MCP
- https://github.com/MiniMax-AI/OpenRoom
- https://github.com/MiniMax-AI/Mini-Agent

OPENROUTER (model + docs)
- https://openrouter.ai/minimax/minimax-m2.7
- https://openrouter.ai/docs/quickstart
- https://openrouter.ai/docs/api/reference/overview
- https://openrouter.ai/docs/guides/community/openai-sdk
- https://openrouter.ai/docs/guides/features/tool-calling

LITELLM (docs + security)
- https://docs.litellm.ai/docs/providers/minimax
- https://docs.litellm.ai/blog/security-update-march-2026
- https://docs.litellm.ai/release_notes/v1.83.0/v1-83-0

LANGCHAIN / LLAMAINDEX / AUTOGEN (framework docs)
- https://docs.langchain.com/oss/python/integrations/providers/minimax
- https://docs.langchain.com/oss/python/integrations/chat/openrouter
- https://developers.llamaindex.ai/python/framework-api-reference/llms/minimax/
- https://github.com/run-llama/llama_index/releases
- https://microsoft.github.io/autogen/stable//reference/python/autogen_ext.models.openai.html

ARTIFICIAL ANALYSIS (third-party evaluation)
- https://artificialanalysis.ai/articles/minimax-m2-7-everything-you-need-to-know

COMMUNITY SIGNAL (GitHub issues)
- https://github.com/RooCodeInc/Roo-Code/issues/11981
- https://github.com/cline/cline/issues/9972
- https://github.com/anomalyco/opencode/issues/18068
- https://github.com/open-webui/open-webui/issues/22843

COMMUNITY SIGNAL (Reddit)
- https://www.reddit.com/r/LocalLLaMA/comments/1rxwcda/benchmarked_minimax_m27_through_2_benchmarks/
- https://www.reddit.com/r/LocalLLaMA/comments/1rznhkj/i_understand_the_disappointment_if_minimax_27/
- https://www.reddit.com/r/ChatGPT/comments/1ry7t5r/chinese_alignment_in_minimax_m27/

LICENSING / OPEN WEIGHTS TIMELINE SIGNAL
- https://huggingface.co/MiniMaxAI/MiniMax-M2.5/discussions/53
- https://huggingface.co/MiniMaxAI/MiniMax-M2.7  (note: access may be gated; attempted fetch returned 401 in this research session)

ADOPTION EXAMPLES (configs / repos)
- https://github.com/bytedance/deer-flow/blob/main/config.example.yaml
- https://github.com/madebyaris/advance-minimax-m2-cursor-rules
- https://github.com/modelscope/sirchmunk
- https://ollama.com/library/minimax-m2.7
```