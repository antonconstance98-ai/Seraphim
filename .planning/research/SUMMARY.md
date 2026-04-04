# Project Research Summary

**Project:** Claude X Codex — v3.0 Adaptive Intelligence
**Domain:** ML-driven self-optimization for multi-model CLI orchestration (Claude Opus 4.6 + Codex GPT-5.4 + MiniMax M-2.7)
**Researched:** 2026-04-03
**Confidence:** MEDIUM-HIGH

## Executive Summary

This milestone adds an adaptive self-optimization layer on top of a working multi-model hook pipeline. The system already has 18 hook scripts, JSONL token logging, and a global dashboard — the v3.0 goal is to make the pipeline learn from its own routing decisions and auto-tune thresholds, review depth, and model selection over time. Research across RouteLLM, Not-Diamond, Martian, and academic literature confirms that no existing router fits a single-user CLI context with implicit signals, stateless hooks, and code-review rather than Q&A tasks. This system is genuinely novel at its scope, which means the architecture must be built from first principles rather than adapted from a service-oriented framework.

The recommended approach is a three-layer design: (1) a passive decision logger appended to the PostToolUse and Stop hook chains, (2) an offline statistical analyzer run at SessionStart, and (3) an atomic config writer called only when the analyzer's confidence gate passes. The ML model itself should be weighted running statistics — not a neural network. At 50–200 hook events per day, neural networks overfit trivially. Interpretable threshold rules on 30+ events are more reliable, auditable, and debuggable than a 10-parameter model trained on 50 examples. The existing `simple-statistics` and `ml-regression` pure-JS libraries cover everything needed without native binaries or Python.

The primary risk is not the ML model failing — it is the system learning to satisfy its metrics rather than its goals (reward hacking), and the system making cascading config adjustments that drift into a broken state. Both are solved by the same safeguard: a confidence gate (minimum 0.8, requiring 30+ events per parameter), hard parameter floor/ceiling enforced in code, an atomic audit trail on every config change, and a freeze mode that bypasses the ML layer entirely. These safeguards must be built in the same phase as the optimizer, not deferred.

## Key Findings

### Recommended Stack

The existing stack (Node.js v22.22.0, `openai@6.33.0`, `@openai/codex` CLI, Claude Code hooks) requires no changes. Three new npm packages are needed: `better-sqlite3@12.8.0` for structured decision log storage with SQL range queries, `simple-statistics@7.8.9` for zero-dependency pure-JS statistics (Z-score, rolling averages, linear regression), and `ml-regression@6.3.0` for polynomial and multivariate regression if the dataset grows to need multi-feature correlation. All three are pure JS or have pre-built binaries for Node 22.

The `node:sqlite` built-in was explicitly rejected — it is still behind `--experimental-sqlite` on Node.js 22 and not suitable for production hook scripts that fire on every tool use. TensorFlow.js was rejected due to 150 MB native binary overhead. `brain.js` was rejected — it is stuck at `2.0.0-beta.24` (last publish ~April 2025).

**Core new technologies:**
- `better-sqlite3@12.8.0`: synchronous SQLite — matches hook execution model; supports JSON columns and window functions for trend queries
- `simple-statistics@7.8.9`: zero dependencies; covers Z-score, percentiles, and linear regression for datasets under 5,000 rows
- `ml-regression@6.3.0`: multivariate regression for multi-feature routing correlation if data volume justifies it
- Node.js v22 built-ins (`fs/promises`, `fetch`): no additional packages needed for file I/O

### Expected Features

**Must have (table stakes — learning infrastructure, P1):**
- `token-log.jsonl` schema extension with `latency_ms`, `accepted`, `dismissed`, `committed` nullable fields (backward-compatible)
- `/gsd:dismiss-last` command — explicit negative signal for false-positive blocks; the only reliable training signal for block quality
- Task-type taxonomy expansion from 4 to 8–12 categories (refactor, explain, security-scan, plan-review, doc-update, test-write, architecture, debug, other)
- Noise profile module (`noise-profile.js`) — per-project rule suppression after 3 consecutive dismissals in 30 days
- Opt-out freeze flag (`"adaptive": false` in settings.json) — safety escape hatch; required before any adaptive logic ships
- Budget guardrail module — 7-day rolling spend; emits downgrade advisory at 80% of $15/day ceiling

**Should have (after P1 data is flowing, P2):**
- Implicit git commit signal — SessionStart looks back at previous session's commits; `"committed": true` appended to matching records
- Per-project routing weights — JSON config keyed by project path prefix
- Feedback loop dashboard panel — dismiss rate, false-positive trend, routing efficiency over time

**Defer to v4+ (needs 500+ labeled records first):**
- Review quality confidence score (breaking schema change — `block_summary` string replaced with structured JSON)
- ML-trained task classifier (only if keyword/regex accuracy falls below 85%)
- Cost-quality Pareto frontier visualization

### Architecture Approach

The system adds three new components that sit alongside the existing 18 hooks without modifying them. `decision-logger.js` runs last in the PostToolUse and Stop chains, parsing advisory text emitted by preceding hooks to infer routing and scan decisions — no existing hook is touched. `ml-analyzer.js` runs at SessionStart, reads the accumulated decision log, computes weighted statistics per tunable parameter, and writes `recommendations.json`. `config-writer.js` is called by the analyzer only when confidence reaches 0.8 — it applies changes atomically (write-then-rename) and appends every change to `adjustment-log.jsonl`. If any new component crashes, the existing 18 hooks continue working exactly as before.

**Major components:**
1. `decision-logger.js` — append-only per-turn signal capture; fail-silent always; 5s timeout; no config writes ever
2. `ml-analyzer.js` — offline statistics only; runs once per session at SessionStart; 10s timeout; writes `recommendations.json` and advisory stdout
3. `config-writer.js` — atomic write-then-rename; safety bounds per parameter; append-only audit trail in `adjustment-log.jsonl`
4. `decision-log.jsonl` (per-project + global) — structured training signal with config snapshot at time of decision for drift detection
5. `defaults.json` — cold-start priors with confidence 0.5; never auto-applied; written at install time, never overwritten

### Critical Pitfalls

1. **Metric gaming / reward hacking** — The optimizer learns to satisfy the tracked proxy (block rate, acceptance rate) rather than actual routing quality. Prevention: composite hard-to-game metrics; holdout set of known-good decisions never used for training; never reward the absence of blocks — reward block accuracy. Must be resolved before any ML model touches live routing decisions.

2. **Catastrophic auto-adjustment** — Small individually reasonable config changes compound into a broken system. Prevention: hard parameter clamps (scan threshold: 1–100; compress tokens: 500–10,000; compress context pct: 50–95); maximum 15% change per parameter per cycle; smoke test with rollback after every auto-adjustment.

3. **False positive block suppression** — Implicit silence (Claude continued without override) used as a negative label teaches the model to suppress Codex blocking for all conversational-adjacent tasks. Prevention: require explicit `/gsd:dismiss-last` for any training example involving a block; never infer label from silence; keep routing model and block quality model as separate training sets.

4. **Data quality / garbage signals** — MiniMax and Codex cached-token pricing use different denominators (flagged HIGH severity in Phase 10 adversarial review). Prevention: per-provider cost normalization tables; define ground truth vs. proxy signals before collection begins; data validation pass before each training run.

5. **Cold start degradation** — ML routing deployed before sufficient data causes random decisions. Prevention: static heuristic router active by default; ML activation gated behind 30+ events per parameter; confidence shown to user as "N/30 events collected"; auto-apply never enabled in first 30 days.

6. **Synchronous hook latency** — ML inference in the PostToolUse hot path adds 50–500ms per tool call. Prevention: decision-logger does zero inference (append-only); ml-analyzer runs only at SessionStart; all hook-path logic must be under 5ms in-process — no remote API calls.

## Implications for Roadmap

The dependency graph is clear: you cannot analyze what you have not captured, and you cannot safely auto-apply what you have not validated in isolation. Four sequential phases match the architecture's natural layers.

### Phase 1: Decision Capture Infrastructure
**Rationale:** Everything downstream depends on a reliable, structured training signal. This phase has no ML — it is pure logging with no risk of misconfiguration. Validates immediately by running a session and checking file output.
**Delivers:** `decision-logger.js`; `decision-log.jsonl` schema; `defaults.json`; token-log.jsonl schema extension; task-type taxonomy (4 → 12 categories); `/gsd:dismiss-last` command; opt-out freeze flag
**Addresses:** Outcome recording, dismiss tracking, task-type enrichment, opt-out/freeze (all P1 table stakes)
**Avoids:** Cold start (Pitfall 4), data quality corruption (Pitfall 3), false positive suppression trap (Pitfall 6)

### Phase 2: Analysis Engine (read-only, no config writes)
**Rationale:** The analyzer must be built and validated against real Phase 1 data before it is allowed to touch any config file. Running it read-only first eliminates the risk of a calibration bug causing catastrophic adjustment.
**Delivers:** `ml-analyzer.js` with weighted running statistics; cold-start / mixed / full statistics modes; `recommendations.json`; mtime-gated incremental read; noise profile module; budget guardrail module
**Uses:** `simple-statistics@7.8.9`; `better-sqlite3@12.8.0` (migrate from JSONL when query complexity grows)
**Avoids:** Recency bias (Pitfall 5 — 30-day window minimum, anomaly weighting); metric gaming (Pitfall 1 — composite metrics defined here, not in Phase 3)

### Phase 3: Config Writer + Audit Trail
**Rationale:** Highest-risk component — writes to files read by all 18 hooks on every invocation. Must be tested in isolation with synthetic inputs before being wired to automatic triggers.
**Delivers:** `config-writer.js` with atomic write-then-rename; safety bounds enforcement; `adjustment-log.jsonl`; rollback capability; freeze-mode verification; model versioning convention
**Avoids:** Catastrophic auto-adjustment (Pitfall 2 — hard clamps enforced here); complexity creep / debugging impossibility (Pitfall 8 — versioning and audit trail established here)

### Phase 4: SessionStart Integration + Confidence Gate
**Rationale:** Wire the three components together only after each is verified standalone. A broken SessionStart hook delays every session open — it must be robust before registration.
**Delivers:** `ml-analyzer.js` registered in settings.json SessionStart chain (timeout: 10s); confidence gate wiring analyzer to config-writer (threshold: 0.8); routing decision audit log in dashboard; per-project routing weights
**Avoids:** Synchronous hook latency (Pitfall 7 — analyzer is SessionStart, not PostToolUse); cold start gate enforced here

### Phase 5: Feedback Loop Dashboard + Implicit Signals
**Rationale:** These features require Phase 1 data to have been flowing for at least 2 weeks before they are meaningful. P2 features that extend the adaptive layer once the core loop is validated.
**Delivers:** Feedback loop dashboard panel (dismiss rate, false-positive trend, routing efficiency); implicit git commit signal; per-project routing weights visible in dashboard

### Phase Ordering Rationale

- Phase 1 before Phase 2: analysis on zero data is meaningless; real events must accumulate before calibration
- Phase 2 before Phase 3: validating recommendations read-only catches calibration bugs before they can corrupt config files
- Phase 3 before Phase 4: config writer is highest-risk; independent verification required before automatic triggering
- Phase 4 before Phase 5: core loop must be stable before extending the feedback surface
- Phases 1–3 together implement all three layers of the catastrophic-drift guardrail (Pitfall 2)

### Research Flags

Phases needing deeper research during planning:
- **Phase 2:** Window size, anomaly weighting parameters, and confidence calibration depend on actual data distributions that only emerge from Phase 1. Plan Phase 2 after 2+ weeks of Phase 1 data is available. Research should include: what constitutes a "sufficient" anomaly threshold at this data volume, and how to construct the holdout set from scratch.
- **Phase 5:** Implicit git signal via backward-looking SessionStart hook is a novel pattern with untested edge cases (rebases, force-pushes, multi-project sessions). Design review needed before implementation.

Phases with standard patterns (skip research phase):
- **Phase 1:** Append-only JSONL logging and schema extension are established patterns in this codebase
- **Phase 3:** Atomic write-then-rename is a proven POSIX pattern already used in this codebase (`codex-global-aggregator.js`)
- **Phase 4:** SessionStart hook registration follows established settings.json patterns; dashboard extension is incremental

## Confidence Assessment

| Area | Confidence | Notes |
|------|------------|-------|
| Stack | HIGH | All versions live-verified via npm registry; Node.js v22 built-ins tested on this machine; `node:sqlite` experimental status confirmed |
| Features | MEDIUM | Drawn from RouteLLM, Not-Diamond, Martian, and 2025 academic literature; no direct prior art for single-user CLI adaptive orchestration at this scope; boundaries are research-backed inferences |
| Architecture | HIGH | Based on live inspection of all 18 hook scripts and settings.json; data flow matches advisory-text-parsing pattern already used in this codebase |
| Pitfalls | HIGH (structural: reward hacking, catastrophic drift — backed by METR 2025 and Google Rules of ML); MEDIUM (hook-specific: extrapolated from hooks behavior + online learning literature) |

**Overall confidence:** MEDIUM-HIGH

### Gaps to Address

- **Reward function weights:** The exact composition of the composite routing quality metric (human override rate, task success, latency penalty, cost penalty) requires empirical calibration. Flag for validation after 2+ weeks of Phase 1 data flows.
- **Holdout set construction:** Research specifies a holdout set of known-good routing decisions is needed to detect metric gaming. With no labeled history at Phase 2 start, manually label 20–30 representative decisions as the initial holdout.
- **MiniMax cached-token cost semantics:** Phase 10 adversarial review flagged this as HIGH severity. Cost normalization tables must be verified against current provider pricing before any budget guardrail logic is written in Phase 2.
- **Advisory text format contracts:** `decision-logger.js` infers decisions by parsing advisory text from preceding hooks. This parsing is brittle if hook output format changes. Advisory text format contracts for each signal-producing hook should be documented and frozen before Phase 1 ships.

## Sources

### Primary (HIGH confidence)
- Live codebase: `/home/alucard/.claude/hooks/` — all 18 hook scripts inspected for data flow and advisory text patterns
- Live settings: `/home/alucard/.claude/settings.json` — hook registrations, timeouts, and event types confirmed
- Live data: token-log.jsonl schema verified from live file on this machine
- npm registry (2026-04-03): `simple-statistics@7.8.9`, `ml-regression@6.3.0`, `better-sqlite3@12.8.0`, `brain.js@2.0.0-beta.24` versions live-verified
- METR (2025): "Recent Frontier Models Are Reward Hacking" — https://metr.org/blog/2025-06-05-recent-reward-hacking/
- Google Rules of Machine Learning — https://developers.google.com/machine-learning/guides/rules-of-ml
- Phase 10 adversarial review (this project): MiniMax token cost semantic mismatch — HIGH severity

### Secondary (MEDIUM confidence)
- RouteLLM (ICLR 2025): preference-label routing; 85% cost reduction claim — https://arxiv.org/pdf/2406.18665
- Confidence-Aware Routing (2025): F1 0.61 → 0.82 with confidence scores — https://arxiv.org/abs/2510.01237
- Datadog Bits AI: false-positive filtering with human feedback — https://www.datadoghq.com/blog/using-llms-to-filter-out-false-positives/
- vLLM Signal-Decision Architecture (2025): signal/decision separation for interpretability — https://blog.vllm.ai/2025/11/19/signal-decision.html
- Sifting the Noise (2025): LLM agents for SAST FP filtering; 92% FP → 6.3% — https://arxiv.org/abs/2601.22952
- better-sqlite3 GitHub: synchronous API rationale, Node.js 22 support — https://github.com/WiseLibs/better-sqlite3

### Tertiary (LOW confidence)
- WebSearch: "Node.js machine learning libraries 2026 lightweight local inference" — candidate identification only; validated by npm cross-check
- Online learning recency bias and window sizing — general literature; parameters require empirical calibration on this dataset

---
*Research completed: 2026-04-03*
*Ready for roadmap: yes*
