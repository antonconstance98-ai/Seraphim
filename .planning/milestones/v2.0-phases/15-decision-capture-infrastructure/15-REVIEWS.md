# Phase 15 — Cross-Model Plan Review

**Reviewed:** 2026-04-04T04:33:56.923Z
**Models:** Round 1: gpt-5.4 (constructive), Round 2: minimax-m2.7 (adversarial)
**Plans reviewed:** 15-01-PLAN.md, 15-02-PLAN.md, 15-03-PLAN.md
**Review type:** Multi-round cross-model (Codex constructive + MiniMax adversarial)

## Findings

=== Round 1 (constructive) ===
[PLAN 01] [SEVERITY: HIGH] Missing verification commands. The visible plan defines deliverables and truths but does not specify concrete commands an executor should run to validate them, such as invoking the modified hooks with fixture stdin, checking `token-log.jsonl` for `model_latency_ms`/`hook_latency_ms`, checking `.planning/decision-log.jsonl` for assembled records, and confirming state-file creation/cleanup in `.planning/.hook-state/`.

[PLAN 01] [SEVERITY: HIGH] Task actions are incomplete/vague in the provided plan body. The only visible task starts with “Create `hook-signal.js` ... and `decision-logger.js` ...” but the actual implementation steps are truncated, so an executor does not have a complete ordered action list for modifying `codex-review-gate.js`, `minimax-post-scan.js`, and `~/.claude/settings.json`.

[PLAN 01] [SEVERITY: MEDIUM] Requirements coverage gap for `DCAP-03`. The plan states “12-category task taxonomy” but the visible content does not map all required categories to concrete source events or specify how PostToolUse vs Stop vs null `toolName` cases are classified beyond one fallback rule. An executor may need the full category definitions and mapping rules to implement consistently.

[PLAN 01] [SEVERITY: MEDIUM] Requirements coverage gap around hook registration. `decision-logger.js` is listed as an artifact and `~/.claude/settings.json` is in `files_modified`, but the visible tasks do not explicitly say where `decision-logger.js` must be inserted in the `PostToolUse` and/or `Stop` hook chains, nor in what order relative to `codex-token-logger.js`, `minimax-post-scan.js`, and `codex-review-gate.js`.

[PLAN 01] [SEVERITY: MEDIUM] Dependency risk on project-local state path resolution. The plan requires hooks under `~/.claude/hooks/` to write state into project `.planning/.hook-state/`, but the visible actions do not explicitly define how project root is derived for both `PostToolUse` and `Stop` payloads or what happens when `cwd` is missing/non-project. That is implementation-critical for the shared state contract.

[PLAN 01] [SEVERITY: LOW] File conflict risk on `~/.claude/settings.json` cannot be fully assessed because parallel plans are not provided. This plan modifies a shared global hook registration file, which is a common collision point across waves and should be checked against other plans before execution.

---

=== Round 2 (adversarial) ===
[CONCERN] [SEVERITY: HIGH] {The plan assumes the deterministic `event_id` can be derived consistently from every hook’s stdin payload, but the payload shapes differ materially between `Stop` and `PostToolUse`. If the derivation uses fields that are absent, reordered, normalized differently, or mutated across hook stages, upstream signals and downstream aggregation will silently de-correlate. Result: valid JSONL everywhere, but the decision logger reads the wrong state or no state at all.}

[CONCERN] [SEVERITY: HIGH] {Per-event state in `.planning/.hook-state/` assumes every hook runs in the same project root and that `cwd` is trustworthy. If a hook is triggered from a subdirectory, symlinked path, different repo root, or a tool operating outside the project tree, signals will be written to one state directory and read from another. This is the cleanest path to production-grade silent failure.}

[CONCERN] [SEVERITY: HIGH] {Append-only JSONL state files do not solve concurrency. Multiple hooks for the same event can race on creation, append, read-before-flush, cleanup, or partial writes. The plan says “append-only” as if that eliminates corruption, but it only avoids read-merge-write conflicts. It does not prevent interleaving, torn records, duplicate records, or logger reads that occur before upstream writes complete.}

[CONCERN] [SEVERITY: HIGH] {The hook chain ordering is not guaranteed to produce complete state before `decision-logger.js` executes unless the settings registration is updated exactly and preserved forever. Any reordering, future hook insertion, timeout, or partial failure yields structurally valid but semantically incomplete decision records. That satisfies “decision-log.jsonl contains a structured record” while being practically useless for downstream analysis.}

[CONCERN] [SEVERITY: HIGH] {Cleanup is underspecified. If `cleanupHookState` runs too early, late readers lose data. If it runs too late or not at all, state accumulates indefinitely, and later `event_id` collisions can contaminate new events with stale signals. At 2 AM this becomes either disk growth or misattributed review decisions, both of which look like legitimate telemetry until someone audits raw files.}

[CONCERN] [SEVERITY: HIGH] {The plan relies on a deterministic `event_id` without proving collision resistance. If the ID is built from low-entropy fields like `session_id`, `tool_name`, `file_path`, or content hashes of truncated payloads, repeated edits to the same file in one session can alias. That turns separate review/scan decisions into one blended state file and corrupts data integrity without throwing errors.}

[CONCERN] [SEVERITY: HIGH] {The requirement “After any hook that makes a model call, the token-log.jsonl record for that call includes `model_latency_ms` and `hook_latency_ms`” is technically fragile because existing hooks appear to emit multiple token-log records per run. Executors can satisfy this by adding latency to one code path and miss early exits, retries, exception branches, or secondary provider calls. The schema will look upgraded while coverage is incomplete.}

[CONCERN] [SEVERITY: HIGH] {The plan depends on upstream hooks writing signals on “ALL early-exit paths” in `minimax-post-scan.js`, but those paths are exactly where implementers forget to preserve invariants. One missed `return` before signal write yields scan absence that is indistinguishable from “scan not triggered,” which poisons downstream labels. This is the simplest executor misinterpretation in the entire plan.}

[CONCERN] [SEVERITY: HIGH] {The Stop hook has a 300s timeout in current settings, but the plan also says hook timeout matches project convention 10s. That is an unresolved contract conflict. An executor can obey either statement and be “correct” locally while breaking production behavior. If they force 10s on a model-calling review gate, timeouts become common and telemetry turns into selective underreporting.}

[CONCERN] [SEVERITY: HIGH] {Writing shared state into project `.planning/` creates a data integrity and privacy problem. Hooks are now coupling runtime telemetry to repo-local files that may be git-visible, backed up, synced, inspected by other tooling, or accidentally committed. Review verdicts, scan findings, prompts, filenames, or tool results can leak into project history or external storage.}

[CONCERN] [SEVERITY: HIGH] {The plan assumes `stderr` logging is enough error handling. It is not. Hook ecosystems commonly ignore stderr, suppress it, or bury it in tool logs nobody reads. That means the system still silently fails in practice while meeting the plan’s “NOT silently swallowed” wording on paper. This is a classic technically-satisfied, operationally-broken requirement.}

[CONCERN] [SEVERITY: HIGH] {`decision-log.jsonl` append semantics are not protected against process crashes or concurrent appenders. JSONL is only safe if every writer guarantees atomic line writes under the actual filesystem and runtime behavior. Node’s `appendFileSync` is not a magic transaction boundary across separate processes. Corrupt trailing lines at 2 AM are inevitable under abnormal termination.}

[CONCERN] [SEVERITY: MEDIUM] {The 12-category taxonomy is presented as a stable truth even though the existing review gate only classifies into four types: `security`, `test-gen`, `bulk-ops`, `feature`. That gap invites arbitrary executor inference. The simplest misread is mapping old categories into new ones with lossy heuristics that technically populate the field but destroy signal quality.}

[CONCERN] [SEVERITY: MEDIUM] {The fallback rule “null `toolName` + `PostToolUse` => `explain`” is dangerously convenient. It converts malformed, unexpected, or incomplete payloads into a legitimate-looking task type instead of surfacing them as unknown/broken. This hides instrumentation defects behind a plausible class label.}

[CONCERN] [SEVERITY: MEDIUM] {The plan assumes `MultiEdit` uses top-level `file_path` because one live file was “confirmed.” That is brittle. If payload shape varies by tool version, provider, or edge path, post-scan will silently skip real edits and still write `scan_triggered=false`, creating false negatives that look intentional.}

[CONCERN] [SEVERITY: MEDIUM] {“No git diff available” is treated as an early-exit case, but many real production contexts lack a clean git diff: detached HEAD, rebases, shallow clones, generated files, hooks firing before index updates, or operations outside tracked files. The plan converts these operational realities into absent scans, meaning the noisiest environments produce the weakest telemetry.}

[CONCERN] [SEVERITY: MEDIUM] {State files in `.planning/.hook-state/` assume the directory always exists and is writable from user-scope hooks. If the repo is read-only, permissions change, `.planning` is missing, or hooks run before the project is fully initialized, signal writes fail and downstream logging degrades to partial records. This is especially likely in fresh clones and CI-like local setups.}

[CONCERN] [SEVERITY: MEDIUM] {The plan says event state is keyed by deterministic `event_id` from stdin payload, but payloads can contain large `tool_result` or `content` blobs. If the event ID is built from serialized payload content, executor implementations can accidentally create huge hashing overhead, normalize line endings inconsistently, or include nondeterministic fields. If they exclude too much, collisions rise. Either direction is bad.}

[CONCERN] [SEVERITY: MEDIUM] {The distinction between `model_latency_ms` and `hook_latency_ms` is not operationally defined. Does hook latency include model latency, preprocessing, file I/O, retries, queueing, JSON parsing, or only wrapper overhead? Different executors will instrument different boundaries and generate incomparable data that still passes schema checks.}

[CONCERN] [SEVERITY: MEDIUM] {The plan relies on “registered PostToolUse (Bash/Edit/Write/MultiEdit/Agent/Task) or Stop hook chain completes” as if hook completion means semantic completion. It does not. A hook can complete after skipping work, timing out an internal subprocess, emitting degraded output, or writing stale cached state. Downstream analysis will treat those as first-class training signals.}

[CONCERN] [SEVERITY: MEDIUM] {Modifying multiple user-scope hooks plus `~/.claude/settings.json` creates a cross-project blast radius. A bug in this repo’s decision capture infrastructure can break or distort hook behavior for unrelated repositories, because the hooks live in the shared home directory. The plan treats this as local infrastructure when it is actually user-global.}

[CONCERN] [SEVERITY: MEDIUM] {The key link requirement checks for code patterns like `writeHookSignal(` and `appendFileSync.*decision-log\.jsonl`. That invites compliance theater. An executor can satisfy the grep patterns while writing incomplete, wrong, or unreachable logic. The review criteria reward superficial linkage, not correctness under execution.}

[CONCERN] [SEVERITY: MEDIUM] {The requirement to capture “every routing and review decision” is practically false from the start because only specific hooks and tool matchers are registered. Anything outside `Bash|Edit|Write|MultiEdit|Agent|Task`, any disabled hook, any aborted tool, or any alternate execution path falls outside coverage. The promise of completeness is already broken in the specification.}

[CONCERN] [SEVERITY: LOW] {Schema versioning is included, but forward compatibility is mostly fictional without a migration strategy, consumer version negotiation, or handling for mixed-version state files. This field will make everyone feel safer while doing very little during actual schema drift.}

[CONCERN] [SEVERITY: LOW] {Using project-local state instead of `/tmp` is framed as a robustness improvement, but it trades volatility for persistence without addressing lifecycle. You remove one class of disappearance and add stale-state contamination, repo pollution, and accidental disclosure.}

[CONCERN] [SEVERITY: LOW] {The plan treats `readAdaptiveFlag returns null on error` as a correctness improvement, but null is just another ambiguous state unless all consumers distinguish `false`, `null`, and “missing” consistently. In practice, one caller will coerce null and behavior will diverge silently.}

[CONCERN] [SEVERITY: LOW] {The objective says this “unblocks all downstream analysis,” which is almost certainly false. If telemetry quality is inconsistent for even a minority of events, downstream phases will build analysis on contaminated labels and missingness patterns. The infrastructure may unblock work administratively while making the results statistically misleading.}

## Verdict

BLOCKED_HIGH_SEVERITY
