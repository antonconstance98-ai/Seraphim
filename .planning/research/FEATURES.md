# Feature Research

**Domain:** ML-driven self-optimization for multi-model AI orchestration (Claude Opus + Codex GPT-5.4 + MiniMax M-2.7)
**Researched:** 2026-04-03
**Confidence:** MEDIUM — routing optimization patterns drawn from RouteLLM, Not-Diamond, Martian, and 2025 academic literature; false-positive filtering from Datadog and vLLM Signal-Decision Architecture; no direct prior art for single-user CLI adaptive orchestration at this exact scope.

> **Milestone scope:** v3.0 — Adaptive Intelligence (ML-based auto-tuning). The existing v2.0
> infrastructure (static routing rules, dual review gate, multi-round plan review, PostToolUse
> scanner, JSONL token logging, global dashboard) is already shipped and forms the dependency
> base for every feature listed here. This file covers only what needs to be built for
> adaptive/self-improving behaviour.

---

## Existing Infrastructure (v2.0 Contract)

Everything in this section is already live. New features depend on it, not replace it.

| Component | What It Provides | Key Data Available |
|-----------|-----------------|-------------------|
| `token-log.jsonl` per project | One record per model call | `model`, `task_type`, `verdict`, `block_summary`, `tokens.*`, `cost_usd`, `timestamp`, `session_id` |
| Global dashboard (`dashboard.html`) | Aggregated cost + catch-rate view | Cross-project summaries, per-session stats, BLOCK issue log |
| Dual review gate (Stop hook) | Codex + MiniMax parallel review | Pass/block decisions, summaries |
| Multi-round plan review | Round 1 constructive, Round 2 adversarial | Structured feedback per round |
| PostToolUse scanner | Bug/security scan after every file write | Per-tool verdict + block_summary |
| Three-tier fallback | Codex CLI → MiniMax API → user prompt | Fallback events, reasons |
| Static routing config | Advisory rules, Opus decides | `~/.claude/settings.json` routing weights |

---

## Feature Landscape

### Table Stakes (Users Expect These)

Self-optimizing orchestration systems are expected to provide these. Missing any of them makes the
system feel manual and not meaningfully different from static rule-based routing.

| Feature | Why Expected | Complexity | v2.0 Dependency | Notes |
|---------|--------------|------------|-----------------|-------|
| Outcome recording — per-call verdict stored with context | All downstream learning depends on having labeled examples. Without it, there is nothing to optimize against. | LOW | `token-log.jsonl` schema already has `verdict` and `block_summary`; needs `accepted` / `dismissed` / `no_action` enrichment | Research: RouteLLM, Not-Diamond, and Martian all require some form of outcome label per routed call. Even thumbs-up/thumbs-down suffices (Wang et al. 2025). |
| Routing metrics collection — latency, cost, task-type per call | Metrics are the raw material for every optimization decision. Without them, the system cannot know which routes are expensive, slow, or low-quality. | LOW | `cost_usd` and `tokens.*` already logged; add `latency_ms` field and enrich `task_type` taxonomy | Martian and LiteLLM routing both track cost + latency + quality score as the three primary axes. |
| False-positive tracking — user can dismiss a review flag as noise | The most painful UX failure in existing static review gates is flag fatigue. Providing a dismiss path creates the training signal needed to suppress recurring noise without breaking the hook pipeline. | MEDIUM | PostToolUse scanner and dual review gate already emit `block_summary`; need a dismiss command (e.g. `/gsd:dismiss-last`) that appends `"dismissed": true` to the relevant JSONL record | Datadog Bits AI: "users can confirm or correct the classification...this human feedback helps refine performance over time." Same pattern works for CLI — thumbs-down = dismiss. |
| Task-type classification enrichment | Routing quality degrades when every call is logged as `"review"`. The ML layer needs richer labels (refactor, explain, security-scan, plan-review, etc.) to learn which models excel at which tasks. | LOW | `task_type` field exists but has only 4 values; expand taxonomy to 8–12 task categories matching actual hook trigger contexts | Not-Diamond trains on task-type + evaluation score pairs. RouteLLM uses MMLU/MT-Bench task categories. |
| Routing decision audit log | Users must be able to see why a call was routed to a specific model. Without explainability, adaptive changes feel opaque and erode trust. | LOW | Dashboard already shows BLOCK issue log; extend it with routing decision column (rule name, confidence score, why this model) | vLLM Signal-Decision Architecture: separates signal extraction from routing decisions explicitly to maintain interpretability. |
| Opt-out / freeze mode | Self-optimization must be pauseable. If the system starts making poor routing decisions (e.g. after a bad batch of training data), the user needs one command to revert to static rules. | LOW | Static routing config in `~/.claude/settings.json` is the freeze target; add `"adaptive": false` flag | Standard safety requirement for any auto-tuning system. Not specific to LLM routing. |

### Differentiators (Competitive Advantage)

These features go beyond what any existing router (RouteLLM, Not-Diamond, Martian) provides out of
the box, because they are specific to the code-review + CLI-hook execution context of this system.

| Feature | Value Proposition | Complexity | v2.0 Dependency | Notes |
|---------|-------------------|------------|-----------------|-------|
| Noise profile per review rule — system learns which MiniMax / Codex flags are recurring false positives for this codebase | Reduces alert fatigue without requiring the user to manually whitelist patterns. The system builds a per-project suppression model from dismiss history. | MEDIUM | Requires dismiss tracking (table stakes above) + per-project rule frequency analysis; no new infrastructure | Datadog's approach: confidence badges per vulnerability type; suppression rules trained from triage history. Equivalent: suppress a rule after N consecutive dismissals in the same project. |
| Implicit outcome signal from git commits — if Claude's output was committed without further edits, that is a positive training signal | Eliminates the need for explicit thumbs-up feedback for the majority of sessions. Leverages existing git history as ground truth. | MEDIUM | Requires a PostSessionEnd hook that checks `git log --since="session_start"` for commits touching files Claude edited; append `"committed": true` to relevant session records | RouteLLM uses Chatbot Arena preference data as its training signal. Equivalent implicit signal in a coding context: the user accepted and committed the output. Under-explored in current literature. |
| Per-project routing weights — high-security projects route more calls to the adversarial (MiniMax) reviewer, low-complexity projects reduce review depth | Cost and review depth are not uniform across projects. Self-optimization should adapt routing intensity to each project's observed risk profile, not apply global defaults. | MEDIUM | Project identity is already inferred from JSONL file path; routing config in `settings.json` could be keyed by project path prefix | Not-Diamond supports custom per-use-case router training. This applies the same principle at project-boundary granularity without retraining a model. |
| Trend-based budget guardrail — system reduces model tier (e.g. swaps GPT-5.4 → GPT-5.4-mini) automatically when 7-day spend approaches the $15/day ceiling | Prevents end-of-month cost surprises. Existing budget constraint ($15/day) is a hard project requirement; adaptive downgrade makes it self-enforcing. | MEDIUM | 7-day rolling cost is computable from existing `token-log.jsonl`; needs a `budget-guardrail.js` module consumed by the routing decision layer | Martian's cost-quality trade-off slider is the commercial equivalent. Adaptive budget guardrails are the single-user CLI equivalent. |
| Review quality score — numerical confidence score (0–1) stored per review verdict, used to weight training signal | Binary BLOCK/ALLOW is too coarse for meaningful learning. A confidence-weighted signal lets the system learn that a low-confidence ALLOW is worth tracking, not just confirmed BLOCKs. | HIGH | Requires prompting MiniMax/Codex to return a structured `{"verdict": "BLOCK", "confidence": 0.87, "summary": "..."}` JSON response instead of free text; backward-incompatible schema change | Confidence-Aware Routing paper (2025): F1 improved from 0.61 to 0.82 and FP rate dropped to 0.09 when confidence scores were used to route borderline cases to human review. |
| Feedback loop dashboard panel — shows dismiss rate, false positive rate trend, routing efficiency over time | Makes the adaptive system's behaviour visible. Without this, the user cannot tell if the system is actually improving or drifting. | MEDIUM | Extends existing `dashboard.html`; needs dismiss + noise profile data from above features | Standard requirement for any production ML system: "you can't improve what you can't measure." |

### Anti-Features (Do Not Build)

| Feature | Why It Seems Good | Why Problematic | What to Do Instead |
|---------|-------------------|-----------------|-------------------|
| Full online ML model retraining (fine-tuning a classifier on local JSONL data) | "Real" ML; matches academic RouteLLM architecture | Training a local classifier requires 100s–1000s of labeled examples to generalise. At single-user CLI scale, the data volume will not be reached for months. Premature ML adds sklearn/torch dependency, a training pipeline, and model versioning — all for a model that predicts from 40 examples. | Use rule-based suppression (N consecutive dismissals → suppress rule) and weighted moving averages for budget guardrails. These are analytically correct at this data volume. Graduate to ML training only when JSONL record count exceeds 500 labeled outcomes. |
| Fully automatic routing without human override | Removes friction; feels like a smart assistant | At this project's scope (one user, CLI, creative/architectural work), fully automatic routing removes the user's ability to course-correct bad routing decisions quickly. The existing design principle — Opus always remains the orchestrator — is violated if routing becomes fully automatic. | Keep all routing advisory. Adaptive system proposes route changes; Opus (and the user) approve via config update. Auto-apply only for budget guardrails where the constraint is a hard cap, not a quality judgement. |
| Embeddings-based semantic router (sentence-transformers, vector similarity) | Matches prompts to known task types semantically; more robust than regex | Requires a local embedding model (100 MB–1 GB), GPU or slow CPU inference, and a vector store. For the task-type taxonomy needed here (8–12 categories), a keyword-rule classifier with 10–20 patterns per category achieves 90%+ accuracy with zero infrastructure. | Keyword + regex task classifier per hook event type. Hook event type alone (PostToolUse vs Stop vs UserPromptSubmit) narrows the task space to 3–4 categories before any text analysis. |
| Human-preference dataset collection (asking the user to rate every output) | More training signal; academic best practice | Annotation burden destroys the UX. Even Chatbot Arena (RouteLLM's training source) notes that labelling fatigue limits dataset size. For a CLI tool used by one person, any UI requiring explicit per-call rating will be ignored within days. | Implicit signals only: git commit presence (positive), dismiss command (negative), session abort without commit (weak negative). Zero annotation burden on the user. |
| Cross-project global routing model | One model for all projects; simpler than per-project models | Different projects have radically different risk profiles (security-critical vs exploratory vs documentation). A global model trained on mixed data will regress toward the mean and underperform on outlier projects. | Per-project routing weights keyed by project path prefix. These are simple JSON config values, not trained models — cheap to inspect, override, and reset. |
| A/B testing framework for routing strategies | Systematic measurement of routing alternatives | A/B testing requires splitting traffic across strategies, tracking outcomes per group, and running statistical significance tests. At single-user CLI scale, this produces no statistically significant results and adds substantial infrastructure. | Sequential comparison: observe baseline metrics for 2 weeks, apply a routing change, observe for another 2 weeks, compare manually. Adequate for a single-user system. |
| Real-time adaptive routing (routing decision updates mid-session) | Most responsive to live session context | Session-level state is not persisted between hook invocations (hooks are stateless processes). Real-time adaptation within a session would require a background daemon with IPC, which contradicts the stateless hook architecture. | Session-to-session adaptation only: routing weights update between sessions based on previous session outcomes logged to JSONL. No mid-session changes. |

---

## Feature Dependencies

```
[Outcome recording — per-call verdict enrichment]
    required-by --> [Noise profile per review rule]
    required-by --> [Implicit signal from git commits]
    required-by --> [Feedback loop dashboard panel]
    required-by --> [Per-project routing weights]

[Task-type classification enrichment]
    required-by --> [Per-project routing weights]
    required-by --> [Routing decision audit log]

[Dismiss tracking (false-positive path)]
    required-by --> [Noise profile per review rule]
    required-by --> [Feedback loop dashboard panel]

[Noise profile per review rule]
    required-by --> [Feedback loop dashboard panel]

[Latency_ms field in token-log.jsonl]
    required-by --> [Routing metrics collection]
    required-by --> [Routing decision audit log]

[Trend-based budget guardrail]
    depends-on --> [7-day rolling cost (computable from existing JSONL)]
    feeds --> [Model tier downgrade (GPT-5.4 → GPT-5.4-mini)]

[Review quality score (confidence 0–1)]
    required-by --> [Noise profile (confidence-weighted suppression)]
    conflicts-with --> [Current free-text block_summary schema — breaking change]

[Opt-out / freeze mode]
    enhances --> [All adaptive features] (safety escape hatch)
```

### Dependency Notes

- **Outcome recording is the root dependency:** Every adaptive feature requires labeled outcome
  data. This must be the first feature built — it is a schema extension to `token-log.jsonl`, not
  new infrastructure. Add `"accepted"`, `"dismissed"`, and `"committed"` nullable boolean fields.

- **Review quality score is a breaking schema change:** If confidence scores are added to hook
  responses, the existing `block_summary` string field must be replaced or supplemented with a
  structured JSON object. All downstream consumers (dashboard, session reporter) need updating.
  Defer this feature until outcome recording and noise profiling are stable.

- **Implicit git signal requires a new hook event or a PostSessionEnd script:** Claude Code does
  not fire a native "session completed" hook that includes git state. A workaround is a
  `SessionStart` hook that looks backward at the previous session's git activity (commits since
  last session start timestamp). This is architecturally similar to the existing
  `codex-cost-reporter.js` pattern.

- **Per-project routing weights depend on task-type enrichment:** Without richer task labels, per-
  project weights cannot distinguish "this project gets more security scanning" from "this project
  makes more plan-review calls." Both look the same with the current 4-value taxonomy.

---

## MVP Definition

### Launch With (v3.0 — this milestone)

Minimum viable adaptive layer. Provides measurable learning without ML infrastructure.

- [ ] `token-log.jsonl` schema extension — add `latency_ms`, `accepted`, `dismissed`, `committed` nullable fields (backward-compatible: existing records read as null)
- [ ] `/gsd:dismiss-last` command — appends `"dismissed": true` to most recent relevant JSONL record; writes human-readable reason to a `dismiss-log.jsonl` sidecar
- [ ] Task-type taxonomy expansion — update hook scripts to classify calls into 8–12 categories (refactor, explain, security-scan, plan-review, doc-update, test-write, architecture, debug, other)
- [ ] Noise profile module (`noise-profile.js`) — reads dismiss history, returns suppression list per project (suppress rule after 3 consecutive dismissals in same project within 30 days)
- [ ] Routing decision log column in dashboard — shows model, task-type, rule that fired, confidence (static placeholder until review quality scores are added)
- [ ] Opt-out freeze flag — `"adaptive": false` in `~/.claude/settings.json` reverts all hooks to v2.0 static behaviour
- [ ] Budget guardrail module (`budget-guardrail.js`) — computes 7-day rolling spend, emits downgrade advisory when >80% of $15/day ceiling reached; Opus decides whether to honour it

### Add After Validation (v3.x)

Add once noise profile and budget guardrail are running and showing measurable changes.

- [ ] Implicit git commit signal — `SessionStart` hook looks back at previous session's git log, appends `"committed": true` to matching session records
- [ ] Per-project routing weights — JSON config keyed by project path prefix; dashboard shows per-project weight diffs from global baseline
- [ ] Feedback loop dashboard panel — dismiss rate, false positive rate trend, routing efficiency over time (cost per accepted output)

### Future Consideration (v4+)

Defer until outcome recording has accumulated 500+ labeled records.

- [ ] Review quality confidence score — structured JSON verdict from MiniMax/Codex with `confidence` field; breaking schema change; requires all consumers updated first
- [ ] ML-trained task classifier — only if keyword/regex classifier accuracy degrades below 85% on observed task distribution
- [ ] Cost-quality Pareto frontier visualisation — plots each model/task-type combination by cost vs catch-rate; useful for tuning but requires rich labeled dataset first

---

## Feature Prioritization Matrix

| Feature | User Value | Implementation Cost | Priority |
|---------|------------|---------------------|----------|
| Outcome recording schema extension | HIGH | LOW | P1 |
| Dismiss command (`/gsd:dismiss-last`) | HIGH | LOW | P1 |
| Task-type taxonomy expansion | HIGH | LOW | P1 |
| Opt-out freeze flag | HIGH | LOW | P1 |
| Budget guardrail module | HIGH | MEDIUM | P1 |
| Noise profile module | HIGH | MEDIUM | P1 |
| Routing decision log in dashboard | MEDIUM | LOW | P2 |
| Implicit git commit signal | MEDIUM | MEDIUM | P2 |
| Per-project routing weights | MEDIUM | MEDIUM | P2 |
| Feedback loop dashboard panel | MEDIUM | MEDIUM | P2 |
| Review quality confidence score | HIGH | HIGH | P3 |
| ML-trained task classifier | LOW | HIGH | P3 |
| Cost-quality Pareto frontier chart | LOW | HIGH | P3 |

**Priority key:**
- P1: Must have for v3.0 launch — these are the learning infrastructure
- P2: Should have in a follow-on patch once P1 data is flowing
- P3: Nice to have, defer to v4+ or until data volume justifies it

---

## Competitor / Prior Art Analysis

| Capability | RouteLLM | Not-Diamond | Martian | LiteLLM Router | This System (v3.0) |
|------------|----------|-------------|---------|-----------------|-------------------|
| Routing signal | Chatbot Arena preference labels | User-uploaded eval dataset | Proprietary | Static rules + cost tiers | Implicit git commits + explicit dismiss |
| False positive suppression | Not applicable (Q&A domain) | Not applicable | Not applicable | Not applicable | Per-project noise profile from dismiss history |
| Budget guardrail | Cost threshold parameter | Cost-quality slider | Cost slider | Budget limits per model | 7-day rolling spend adaptive downgrade |
| Human-in-loop | Not supported | Not supported | Not supported | Not supported | Dismiss command + opt-out freeze |
| Task classification | MMLU/MT-Bench categories | User-defined | Proprietary | Regex rules | Hook event type + enriched 12-category taxonomy |
| Per-project weights | No | Yes (custom router training) | No | No | Yes (JSON config, no model retraining) |
| Explainability | Routing score logged | Not documented | Not documented | Rule name logged | Decision audit log in dashboard |
| Single-user CLI fit | No (service) | No (service) | No (service) | No (proxy) | Yes (hook-native, stateless, no daemon) |

**Gap being filled:** All existing routers are designed for multi-user services with large labeled
datasets. None of them fit a single-user CLI context where: (a) explicit annotation is not viable,
(b) no persistent service can run, (c) the "task" is code review not Q&A, and (d) false positive
suppression is more valuable than raw routing accuracy improvement.

---

## Sources

- RouteLLM framework and 85% cost reduction claims: [GitHub lm-sys/RouteLLM](https://github.com/lm-sys/RouteLLM) and [LMSYS Blog 2024-07-01](https://www.lmsys.org/blog/2024-07-01-routellm/)
- RouteLLM learning from preference data: [arXiv 2406.18665](https://arxiv.org/pdf/2406.18665) — ICLR 2025 published version
- Adaptive LLM routing under budget constraints: [arXiv 2508.21141](https://arxiv.org/abs/2508.21141)
- LLM Routing with Dueling Feedback (bandit feedback approach): [arXiv 2510.00841](https://arxiv.org/html/2510.00841v1)
- Confidence-Aware Routing (F1 0.61 → 0.82): [arXiv 2510.01237](https://arxiv.org/abs/2510.01237)
- Generalised Routing / MoMA framework: [arXiv 2509.07571](https://arxiv.org/abs/2509.07571)
- Not-Diamond meta-model routing: [notdiamond.ai](https://www.notdiamond.ai/) and [Hacker News discussion](https://news.ycombinator.com/item?id=41108787)
- Martian cost-quality trade-off: [VentureBeat model routing article](https://venturebeat.com/ai/why-accenture-and-martian-see-model-routing-as-key-to-enterprise-ai-success)
- Datadog false-positive filtering with LLMs + human feedback: [Datadog blog](https://www.datadoghq.com/blog/using-llms-to-filter-out-false-positives/)
- vLLM Signal-Decision Architecture (signal/decision separation for interpretability): [vLLM Blog 2025-11-19](https://blog.vllm.ai/2025/11/19/signal-decision.html)
- Sifting the Noise — LLM agents for SAST false positive filtering (92% FP → 6.3%): [arXiv 2601.22952](https://arxiv.org/abs/2601.22952)
- Signal and Noise framework for LLM evaluation: [arXiv 2508.13144](https://arxiv.org/html/2508.13144v1)
- IDC — The future of AI is model routing: [IDC Blog](https://www.idc.com/resource-center/blog/the-future-of-ai-is-model-routing/)
- Existing v2.0 infrastructure: verified from live files at `/home/alucard/projects/Claude_X_Codex/.planning/`

---

*Feature research for: ML-driven self-optimization — Claude X Codex v3.0 Adaptive Intelligence milestone*
*Researched: 2026-04-03*
