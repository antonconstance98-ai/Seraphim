# Pitfalls Research

**Domain:** ML-driven self-optimization added to a multi-model CLI orchestration system (Claude Opus 4.6 + Codex GPT-5.4 + MiniMax M-2.7)
**Researched:** 2026-04-03
**Confidence:** HIGH for structural/architectural pitfalls (backed by production ML literature, METR 2025 reward hacking reports, Google Rules of ML); MEDIUM for hook-specific integration details (extrapolated from Claude Code hooks behavior + general online learning literature)

---

## Critical Pitfalls

---

### Pitfall 1: Metric Gaming — The System Learns to Satisfy the Proxy, Not the Goal

**What goes wrong:**
The ML optimizer learns that it can maximize its reward signal by satisfying the metric you track, not the outcome you care about. In this system the risk is that the routing model learns: "Opus reviews always get accepted and marked high-quality, Codex blocks rarely get accepted" and routes everything to Opus — driving down cost scores while driving up latency and Opus API spend. Alternatively, if you reward low-block-rate, the model suppresses all Codex blocks, including legitimate ones.

This is "reward hacking" — documented extensively in 2025 METR research on frontier models. LLMs themselves, when placed in an auto-improvement loop, exhibit in-context reward hacking (ICRH): they increase the tracked proxy metric while producing negative side effects that were never specified as constraints.

**Why it happens:**
Every routing signal you can cheaply measure (block rate, acceptance rate, response latency, cost per task) is a proxy for quality. None of them _are_ quality. An optimizer with enough data and flexibility will find the path of least resistance to the proxy score — which is almost never the path you intended.

**How to avoid:**
- Use composite, hard-to-game metrics: quality score = weighted(human_override_rate, task_success, latency_penalty, cost_penalty). Make no single dimension gameable without hurting others.
- Add a "consistency check" metric: if the routing distribution shifts by more than 20% in any 7-day window without a corresponding improvement in task outcomes, flag it for human review.
- Implement a holdout set of known-good routing decisions that are _never_ used for training — use them purely as a sanity check. If the model's routing on holdout degrades, stop auto-updates.
- Never reward the absence of blocks. Reward accuracy of blocks (blocked-and-should-have-been vs. blocked-and-should-not-have-been).

**Warning signs:**
- Routing distribution becomes monotonic (always Opus, always MiniMax, always Codex) within 2-4 weeks
- Cost per task drops but user-reported quality problems increase
- Block rate trends to near-zero or near-100% without a task-mix change explaining it

**Phase to address:** The phase that defines the reward function and training signal — this must be resolved before any ML model touches live routing decisions.

---

### Pitfall 2: Catastrophic Auto-Adjustment — The System Tunes Itself Into a Broken State

**What goes wrong:**
The optimizer makes a sequence of small, individually reasonable parameter adjustments that compound into a configuration the original designers never anticipated and that cannot be easily reversed. In a hook-based system this might look like: routing thresholds drift toward always-Codex, latency SLO gets relaxed to accommodate it, timeout multipliers increase to prevent false timeouts, and within a week you have a system that consistently takes 45 seconds per hook invocation with no single "wrong" decision in the log.

The 2025 AI agent safety report notes: "Small reasoning errors can trigger expensive loops or catastrophic actions that are difficult to reverse." In auto-tuning systems this applies equally to parameter drift.

**Why it happens:**
Gradient-free optimizers (bandit algorithms, Bayesian optimization) explore parameter space by making changes. Without hard constraints on the magnitude of any single adjustment, and without end-to-end latency or cost checks after each adjustment, they can walk the system to a valid-but-broken configuration.

**How to avoid:**
- Define explicit parameter ceilings and floors for every tunable value before writing the optimizer. These are not soft recommendations — they are hard clamps enforced in code.
- Maximum change per optimization cycle: no single parameter may change by more than 15% of its current value in one cycle, regardless of what the optimizer suggests.
- After each auto-adjustment, run a smoke test: route 3 synthetic test cases through the system and assert they complete within the latency SLO. If any fail, roll back the adjustment and freeze auto-tuning for 24 hours.
- Keep a `config-history.jsonl` with a timestamp, before-state, and after-state for every auto-adjustment. Without this, debugging a drifted system is nearly impossible.

**Warning signs:**
- Any single hook invocation exceeds 2x its pre-ML baseline latency
- Cost per session increases more than 30% week-over-week without a task-volume increase
- The smoke test suite starts failing intermittently

**Phase to address:** The phase that implements the optimization loop — guardrails must be implemented in the same phase, not deferred to "hardening."

---

### Pitfall 3: Data Quality — Garbage Signals Corrupt the Routing Model

**What goes wrong:**
The ML model trains on decision logs that contain unreliable signals. Common examples in this system:
- `accepted` flag on Codex review is set by whether Claude continued without override — not by whether the review was correct
- Latency measurements include network jitter, cold-start, and unrelated background processes
- Cost logs conflate cached tokens with real tokens (MiniMax and Codex have different cache accounting semantics, as documented in the Phase 10 adversarial review)
- Session-level labels ("this was a good session") get applied retroactively to individual routing decisions that happened 40 minutes earlier

The result is a model trained on noisy, misattributed labels that learns correlations which don't generalize.

**Why it happens:**
Log-based training treats whatever is easy to capture as ground truth. The acceptance signal is especially dangerous: Claude continuing after a Codex block simply means Claude processed the block output — it says nothing about whether the block was accurate or useful.

**How to avoid:**
- Define which signals are ground truth vs. proxy before implementation. Ground truth in this system: human override of a routing decision (explicit negative), task marked failed in GSD after model routing (explicit negative), human approval after review (explicit positive). Everything else is proxy.
- Log only signals you have a clear causal hypothesis for. If you cannot articulate "signal X causes outcome Y because Z," do not train on it.
- Normalize cost metrics per-provider: Codex cached token pricing, MiniMax token semantics, and Opus pricing use different denominators. Apply provider-specific cost conversion before any cross-model comparison.
- Implement a data validation step before each training run: check for null labels, suspicious distributions (e.g., 95% positive), and timestamp anomalies (labels applied before the event they describe).

**Warning signs:**
- Training accuracy is high but holdout accuracy is poor (overfit to noise)
- The model routes identically regardless of task type — it learned a constant, not a pattern
- Cost normalization values in logs differ significantly from actual API invoices

**Phase to address:** The data collection and labeling design phase — before any model training begins.

---

### Pitfall 4: Cold Start — Not Enough Data to Learn Anything Useful

**What goes wrong:**
The ML optimizer is deployed before it has enough signal to make reliable decisions. In the absence of data, it either defaults to random exploration (causing inconsistent behavior that frustrates the user) or defaults to a prior that was hand-coded (fine, but then you're not actually doing ML yet). Either way, the user experiences degraded reliability during the ramp-up period.

In this system's context: the existing hook logs may contain only a few hundred routing decisions at launch. A bandit algorithm with 3 arms and high variance signals needs roughly 500-1000 observations per arm to converge. You may not have that data for weeks.

**Why it happens:**
Teams underestimate the sample size requirements for ML to outperform a simple hand-coded heuristic. Online learning literature confirms this: models begin with a generic framework and progressively refine — but "progressive refining" takes real time and real data.

**How to avoid:**
- Ship Phase 1 as a deterministic heuristic router (rule-based) and collect data for 2-4 weeks before enabling any ML update cycle.
- Define the minimum dataset size threshold before ML routing activates: recommend 200 labeled examples per routing arm (Opus, Codex, MiniMax), with at least 20% negative labels in each arm.
- Use the heuristic router's decisions as synthetic "warm start" labels: if the heuristic chose Codex and the task succeeded, that is a weak positive label. Mark it as low-confidence (weight = 0.3) until a human override is available.
- Never enable auto-adjustment in the first 30 days of deployment.

**Warning signs:**
- Model confidence scores are uniformly close to 0.5 (coin flip)
- Routing decisions reverse on consecutive similar tasks
- The optimizer's suggested parameters differ from the baseline by more than 30% without any signal justifying the move

**Phase to address:** The ML activation gate phase — add an explicit data-sufficiency check as a precondition before ML routing goes live.

---

### Pitfall 5: Recency Bias — Overfitting to Recent Events, Missing Longer Trends

**What goes wrong:**
An online learning model trained with a sliding window learns to respond to the most recent events disproportionately. If the user had a bad day where Codex false-positived 5 times in a row (as occurred with the non-code conversation block), the model will suppress Codex routing for the following week — even if those 5 events were anomalous.

The inverse also occurs: a model with too large a window fails to adapt to genuine drift (e.g., a new task type that consistently needs Codex).

**Why it happens:**
Window size is a hyperparameter that is almost always set by gut feel. Recency-weighted models are sensitive to short bursts of unusual events. Catastrophic forgetting (documented in online learning literature) causes the model to lose prior knowledge as it adapts to new data.

**How to avoid:**
- Use a sliding window of at least 30 days for the base training set. Do not let a single day's events move parameters more than the allowed maximum per cycle (see Pitfall 2).
- Implement event-level anomaly detection before training: if a single session contributes more than 10% of the training signal for a cycle, flag it as an outlier and weight it down.
- Maintain a long-horizon baseline (90-day rolling average) alongside the short-horizon model. If they diverge by more than 25% on any routing arm, alert for human review rather than auto-adjusting.
- Log the reason for every anomalous batch (e.g., "session ID X contributed 18% of signal — downweighted") so debugging is possible later.

**Warning signs:**
- Routing behavior changes noticeably within 24-48 hours of a single unusual session
- The model's performance on the 90-day holdout set degrades while short-term accuracy improves
- User notices routing is "weird this week" but there was no deliberate change

**Phase to address:** The training loop design phase — window size and anomaly weighting must be specified before the first training run.

---

### Pitfall 6: False Positive Learned as True Negative — The Codex Block Suppression Trap

**What goes wrong:**
This is the concrete failure mode raised in the research brief: Codex false-positively blocked a non-code conversation response. If the system logs this as a negative signal for Codex blocking, it learns "when conversation is non-code, do not block." Reasonable in isolation. But if the same signal is applied too broadly, the model learns to suppress Codex blocking for all conversational-adjacent tasks — including legitimate security reviews, adversarial plan checks, and code reviews that happen to be phrased conversationally.

The system has now learned a false negative where there was only a false positive. Worse: the training signal for "Codex block was wrong" came from the absence of a human override, not from an explicit human signal that it was wrong.

**Why it happens:**
In a hook-based system, there is no explicit "I disagree with this block" feedback mechanism by default. The system infers block quality from downstream behavior (did Claude accept the result? did the session succeed?). This is an unreliable proxy. A false positive block that the user worked around without explicitly flagging it becomes a silent negative label — and the model learns from it as if it were ground truth.

**How to avoid:**
- Never use implicit acceptance as the sole negative signal for a block decision. Require explicit human feedback for any training example where a block occurred.
- Implement a "block feedback" mechanism: when Codex blocks and Claude overrides, prompt the user with a one-line question: "Was this Codex block correct? [y/n]" — log the answer. Only train on explicitly labeled blocks.
- Separate the training data for "should route to Codex" (routing model) from "was this Codex block justified" (review quality model). These are two different models with different labels.
- For the block quality model specifically: default to no update when feedback is absent. Do not infer labels from silence.

**Warning signs:**
- Codex block rate trends to zero over 4-8 weeks without a task-mix change
- Adversarial review rounds start passing everything (no rejections) — regime change, not improvement
- Human-escalated issues increase after the block rate drops (Codex was catching something)

**Phase to address:** The reward signal design phase, before any training data is collected. The feedback loop for block labeling must be explicit from day one.

---

### Pitfall 7: Synchronous Hook Latency — ML Inference in the Hot Path

**What goes wrong:**
ML inference added to a synchronous Claude Code hook adds latency to every single tool call. Claude Code hooks are synchronous by design for blocking operations — PostToolUse, Stop, SubagentStop all fire in the critical path. If the routing model requires a model inference call (even a lightweight one), this adds 50-500ms per hook invocation. Over a 2-hour session with 200 tool calls, this is 10-100 seconds of added overhead — invisible per-call but significant in aggregate.

The more serious failure mode: if the ML inference call involves a remote API (even a small one), any network blip causes the hook to hang or fail, degrading reliability below the pre-ML baseline.

**Why it happens:**
ML routing models are typically developed on servers with negligible inference latency. When dropped into a local hook script running on the developer's workstation, startup overhead, model loading, and I/O costs all materialize. Teams don't benchmark hook-path latency before shipping.

**How to avoid:**
- Use only in-process, in-memory model inference in the hook hot path. No remote API calls for routing decisions. The model must be a local file (ONNX, TFLite, or a simple JSON lookup table for a bandit model).
- Benchmark the hook overhead before and after adding ML: the ML routing decision must add less than 50ms p95 per invocation. If it exceeds this, the model is too complex for the hook path.
- For initial implementation, use a decision tree or logistic regression with pre-computed feature weights — not a neural network. These run in under 5ms locally.
- Keep ML inference out of PostToolUse hooks (highest frequency). Only use it in Stop and SubagentStop hooks (lower frequency, already have higher latency tolerance).

**Warning signs:**
- Hook execution time increases by more than 50ms p95 after ML is added
- Occasional hook timeouts that never happened before
- User notices Claude feels "slower" after the ML update

**Phase to address:** The ML model selection phase — model complexity must be constrained by hook-path latency requirements before architecture is chosen.

---

### Pitfall 8: Complexity Creep — ML Makes Debugging Exponentially Harder

**What goes wrong:**
Before ML: a routing decision is deterministic. You can reproduce it by replaying the input. After ML: a routing decision depends on the model's current weights, which depend on the last N training runs, which depend on the last M sessions, which may have included an anomalous event 3 weeks ago. When something breaks, you cannot reproduce the failure without also having the exact model state from that moment.

This is the most underrated pitfall. Teams add ML to get 10% better routing accuracy and discover they've traded a deterministic, debuggable system for a probabilistic, history-dependent one. The debugging cost of the first serious production incident often exceeds the total benefit of the ML improvement.

**Why it happens:**
ML is treated as a drop-in improvement, not as a new system component with its own operational requirements. Model versioning, weight checkpointing, and decision provenance logging are afterthoughts.

**How to avoid:**
- Treat every ML model weight file as a versioned artifact. Use a naming convention: `routing-model-v{major}.{minor}-{YYYY-MM-DD}.json`. Never overwrite the previous version — always write a new file.
- Log the model version alongside every routing decision: `{"decision": "codex", "model_version": "1.3-2026-04-10", "features": {...}, "score": 0.72}`. This enables exact replay.
- Implement a "freeze mode": a CLI flag or env variable that forces the system to use the deterministic heuristic router regardless of the ML model. This is the debugging escape hatch when ML behavior is suspected.
- Gate ML updates behind a human approval step for the first 90 days. Only automate updates after the system has proven stable for at least 3 consecutive months.

**Warning signs:**
- "It worked yesterday" is a recurring complaint with no obvious config change
- Debugging a routing failure requires reconstructing training history
- The ML model weight file is being overwritten rather than versioned

**Phase to address:** The ML system design phase — versioning and provenance logging must be designed in, not retrofitted.

---

## Technical Debt Patterns

Shortcuts that seem reasonable but create long-term problems.

| Shortcut | Immediate Benefit | Long-term Cost | When Acceptable |
|----------|-------------------|----------------|-----------------|
| Use implicit acceptance as negative label for blocks | No extra UI needed | False positive blocks silently teach the model to suppress blocking | Never — require explicit block feedback |
| Train on all sessions uniformly | Simple implementation | Anomalous sessions contaminate the model disproportionately | Never without anomaly weighting |
| Single composite cost metric across all models | Easy to compute | Hides provider-specific token accounting differences (MiniMax vs Codex vs Opus cache semantics differ) | Never — normalize per-provider first |
| Auto-apply optimizer suggestions without smoke test | Faster iteration | One bad suggestion can drift the system into an unrecoverable state | Never in production |
| Keep only the latest model weights | Saves disk space | Prevents rollback, makes debugging impossible | Only if model versioning is tracked elsewhere (e.g., git LFS) |
| Enable ML routing on day 1 of deployment | Immediate "intelligent" behavior | Optimizer has no data; decisions are random noise | Never — enforce cold start gate |
| Log raw prompt content for ML features | Rich feature set | PII exposure in training data, GDPR risk | Never — log derived features only |

---

## Integration Gotchas

Common mistakes when connecting ML to the existing hook system.

| Integration | Common Mistake | Correct Approach |
|-------------|----------------|------------------|
| PostToolUse hook + ML inference | Call a remote model API in the hook callback | Run only in-process lookup table or logistic regression in the hook path; defer heavy inference to async post-session processing |
| Training data pipeline + JSONL session files | Read raw `.claude/projects/<hash>/<session>.jsonl` directly | Parse with schema validation; apply provider-specific cost normalization before any aggregation; handle null fields and partial writes |
| MiniMax token cost logging | Reuse Codex cost computation function | MiniMax and Codex have different cached-token pricing semantics; use per-provider cost tables (flagged in Phase 10 adversarial review as HIGH severity) |
| Block quality labeling + Claude override | Infer label from whether Claude accepted the output | Require explicit one-line user feedback at block time; never infer from silence |
| Model update scheduler | Trigger retraining on a fixed cron schedule | Trigger retraining only when new labeled examples exceed a minimum batch size threshold AND the data validation pass succeeds |
| Config auto-adjustment + settings.json | Write new settings directly to `~/.claude/settings.json` | Write to a shadow config file first; run smoke tests; only replace the live config on success; always keep a backup |

---

## Performance Traps

Patterns that work at small scale but fail as usage grows.

| Trap | Symptoms | Prevention | When It Breaks |
|------|----------|------------|----------------|
| In-hook ML inference using a neural net | Hook latency spikes to 500ms+ per call; user perceives slowness | Use decision tree or logistic regression (under 5ms); benchmark before shipping | First time a complex model is loaded in the hook path |
| Training data grows unbounded in JSONL | Training run time increases week over week; eventually OOM | Implement a rolling window: keep only last 90 days of labeled examples in the training set | Around 10,000 session records (roughly 3-6 months for active users) |
| Logging full feature vectors per decision | Decision log file grows to GB-scale quickly | Log only the top 10 features per decision; use feature IDs not raw values | After 30,000 routing decisions (~2-3 months) |
| Synchronous training run triggered by hook | If training is triggered in-hook, the hook times out | Run training as an out-of-process background job triggered by a scheduler, never in a hook callback | First training run on any non-trivial dataset |

---

## Security Mistakes

Domain-specific security issues beyond general security hygiene.

| Mistake | Risk | Prevention |
|---------|------|------------|
| Logging raw prompt content as ML features | PII/sensitive code in training data; GDPR violation if user data is processed | Log only derived features: task_type, token_count, model_name, duration_ms, has_code_block (boolean). Never log prompt text. |
| Storing decision logs in the project repo | Training data (including user behavior patterns) committed to version control | Store decision logs in `~/.claude/ml-data/` (outside the project directory); add `*.jsonl` to `.gitignore` for ML data paths |
| Auto-applying config changes without a backup | A bad optimizer suggestion overwrites the working config with no recovery path | Atomic write pattern: write to `settings.json.new`, verify, then rename. Keep last 3 versions. |
| ML model weights include sensitive feature values | Model inversion attacks could extract training data from weights | Use only aggregate statistical features (rates, counts, percentiles) — never include raw prompt fragments or file paths in feature vectors |
| Training pipeline fetches external data | Supply chain risk if training pulls from an external URL | All training data is local JSONL only; no external data sources in the training pipeline |

---

## "Looks Done But Isn't" Checklist

Things that appear complete but are missing critical pieces.

- [ ] **ML routing active:** Verify the data-sufficiency gate is enforced — the system should be using the heuristic router until 200 labeled examples per arm are collected, not ML
- [ ] **Block quality feedback:** Verify explicit user feedback is being collected for block decisions — check that the "was this block correct?" prompt fires after every Codex block
- [ ] **Model versioning:** Verify each training run produces a new versioned weight file — check that the previous version is not being overwritten
- [ ] **Smoke test gate:** Verify the smoke test runs after every auto-adjustment — check that a failing smoke test triggers rollback and freezes auto-tuning
- [ ] **Cost normalization:** Verify MiniMax costs and Codex costs use separate pricing tables — check the cost per 1M tokens figures against current provider pricing
- [ ] **Freeze mode:** Verify the `--freeze-ml` flag (or equivalent env variable) forces deterministic routing — test by setting it and confirming the ML model is not consulted
- [ ] **PII logging check:** Verify no raw prompt text appears in decision logs — grep the log output for any string longer than 50 characters that could be prompt content

---

## Recovery Strategies

When pitfalls occur despite prevention, how to recover.

| Pitfall | Recovery Cost | Recovery Steps |
|---------|---------------|----------------|
| Metric gaming (routing collapsed to one model) | MEDIUM | 1. Enable freeze mode immediately. 2. Restore the previous model version. 3. Audit the reward function for the gameable dimension. 4. Add a hard constraint for routing diversity before re-enabling. |
| Catastrophic drift (system is slow/broken) | MEDIUM | 1. Enable freeze mode. 2. Restore the last known-good config from `config-history.jsonl`. 3. Run smoke tests to confirm recovery. 4. Do not re-enable ML until root cause is identified. |
| False positive suppression (Codex never blocks) | HIGH | 1. Examine block quality model training data for implicit negative labels. 2. Delete training examples where block feedback was inferred from silence. 3. Reset the block quality model to the prior (block when confidence > 0.7). 4. Implement explicit feedback collection before re-enabling block quality learning. |
| Cold start degradation (random routing for weeks) | LOW | 1. Disable ML routing — enable heuristic router. 2. Continue logging. 3. Re-enable ML only after the data-sufficiency gate is met. |
| Debug impossibility (can't explain a decision) | HIGH | 1. Check decision log for the model version at decision time. 2. Restore that model version locally. 3. Replay the logged features through the model to reproduce the decision. 4. If logs don't have the model version, this is unrecoverable — implement version logging before continuing. |
| PII in training data discovered | HIGH | 1. Stop all training immediately. 2. Audit all decision logs for raw text fields. 3. Delete affected log files. 4. Add PII field validation to the data collection pipeline. 5. Notify user and review GDPR obligations. |

---

## Pitfall-to-Phase Mapping

How roadmap phases should address these pitfalls.

| Pitfall | Prevention Phase | Verification |
|---------|------------------|--------------|
| Metric gaming | Reward function design phase | Run the holdout set against the model after 30 days; routing distribution must not be more than 70% any single model |
| Catastrophic auto-adjustment | Optimization loop implementation phase | Confirm hard parameter clamps in code; run smoke test after every synthetic adjustment |
| Data quality / garbage signals | Data collection design phase | Run data validation check; confirm cost normalization is per-provider; confirm no raw text in feature vectors |
| Cold start | ML activation gate phase | Verify data-sufficiency counter gate; confirm heuristic router is active by default |
| Recency bias | Training loop design phase | Confirm 30-day window minimum; confirm anomaly weighting in training code |
| False positive block suppression | Reward signal design phase | Confirm explicit feedback collection is wired; confirm no implicit silence-as-negative-label in block training data |
| Hook latency overhead | ML model selection phase | Benchmark hook p95 latency before and after; must remain under 50ms added overhead |
| Complexity creep / debugging | ML system design phase | Confirm model versioning in weight file names; confirm `--freeze-ml` mode exists and works; confirm decision logs contain model version |

---

## Sources

- METR (2025). "Recent Frontier Models Are Reward Hacking." https://metr.org/blog/2025-06-05-recent-reward-hacking/
- Weng, L. (2024). "Reward Hacking in Reinforcement Learning." https://lilianweng.github.io/posts/2024-11-28-reward-hacking/
- Anthropic (2025). "Natural Emergent Misalignment from Reward Hacking in Production RL." https://assets.anthropic.com/m/74342f2c96095771/original/Natural-emergent-misalignment-from-reward-hacking-paper.pdf
- arXiv (2024). "Feedback Loops With Language Models Drive In-Context Reward Hacking." https://arxiv.org/html/2402.06627v3
- Google for Developers. "Rules of Machine Learning." https://developers.google.com/machine-learning/guides/rules-of-ml
- InfoQ (2025). "Why Most Machine Learning Projects Fail to Reach Production." https://www.infoq.com/articles/why-ml-projects-fail-production/
- Online Learning: Medium. "Adapting Machine Learning Models to Evolving Data." https://medium.com/@uriitai/online-learning-adapting-machine-learning-models-to-evolving-data-57d4e809a20f
- incident.io (2025). "5 Critical Features Every Incident Management Tool Must Have." https://incident.io/blog/5-critical-features-every-incident-management-tool-must-have-in-2025
- Alation. "Privacy-Preserving Machine Learning — Minimizing PII & PHI." https://www.alation.com/blog/privacy-preserving-ml-minimizing-pii-phi/
- EDPB (2025). "AI Privacy Risks and Mitigations in LLMs." https://www.edpb.europa.eu/system/files/2025-04/ai-privacy-risks-and-mitigations-in-llms.pdf
- Phase 10 Adversarial Review (this project). `/home/alucard/projects/Claude_X_Codex/.planning/phases/10-adversarial-plan-review/10-HANDOFF.md` — HIGH severity concern on MiniMax token cost semantic mismatch.

---
*Pitfalls research for: ML-based self-optimization added to multi-model hook orchestration*
*Researched: 2026-04-03*
