# Phase 11 — Cross-Model Plan Review

**Reviewed:** 2026-04-03T20:03:44.534Z
**Models:** Round 1: gpt-5.4 (constructive), Round 2: minimax-m2.7 (adversarial)
**Plans reviewed:** 11-01-PLAN.md
**Review type:** Multi-round cross-model (Codex constructive + MiniMax adversarial)

## Findings

=== Round 1 (constructive) ===
[PLAN 01] [SEVERITY: HIGH] The plan does not include an explicit verification section or concrete commands. An executor still needs exact checks for: JSON validity of both settings files, hook registration presence, skip behavior on non-code files, skip behavior below `scan_skip_threshold`, advisory stdout on findings, silent success on clean/skip/error, and token-log append behavior.

[PLAN 01] [SEVERITY: MEDIUM] `scan_skip_threshold` is required, but the plan does not define what the threshold measures or how it is computed. An executor would need clarification on whether this is bytes changed, lines changed, characters in diff, or another metric.

[PLAN 01] [SEVERITY: MEDIUM] The non-code skip requirement is still vague at execution level. The plan says markdown, JSON, and planning docs are skipped, but it does not give the executor a definitive allowlist/blocklist of extensions or path rules, which leaves edge cases unresolved.

[PLAN 01] [SEVERITY: MEDIUM] The plan requires logging to `.planning/token-log.jsonl`, but that file is not listed in `files_modified` or `artifacts`. If the hook is expected to create or append to it, the artifact/file-output expectations should say so explicitly.

[PLAN 01] [SEVERITY: LOW] No parallel file conflict is detectable from the material provided, but the review input only shows one plan, so cross-plan conflict analysis is incomplete.

---

=== Round 2 (adversarial) ===
Reviewing the plan text for failure modes and hidden assumptions. I’m focusing on places where the hook will silently skip, misfire, leak data, or satisfy the spec while still failing operationally.
[CONCERN] [SEVERITY: HIGH] {The plan assumes `tool_input.file_path` is always trustworthy and absolute for `Write`, `Edit`, and `MultiEdit`, but that is only an interface claim, not a runtime guarantee. If Claude/Codex emits a malformed payload, a relative path, or a path outside the repo, the scanner can silently skip or scan the wrong file while still exiting 0.}

[CONCERN] [SEVERITY: HIGH] {“Non-code files are skipped” is technically satisfied by extension filtering, but practically broken: many security-relevant files are not conventional code extensions. `Dockerfile`, `Makefile`, `.env.example`, shell snippets without `.sh`, CI YAML, Terraform, SQL migrations, and templated config files can contain critical vulnerabilities and will be silently excluded by design.}

[CONCERN] [SEVERITY: HIGH] {The plan relies on a “trivial edit threshold” to suppress scans, but that threshold is exactly how real vulnerabilities disappear from coverage. One-line edits change auth checks, escape functions, CSP headers, SQL query fragments, SSRF allowlists, and crypto flags. “Small diff” is not a safe proxy for “low risk.”}

[CONCERN] [SEVERITY: HIGH] {Using `git diff HEAD` or staged/untracked fallback is not equivalent to “scan what was just written.” If the worktree is dirty, the diff can include unrelated edits from earlier work, generated files, or user changes. The scanner will attribute findings to the current tool action even when they came from something else, and just as bad, it can miss the actual new content if the file state and diff state diverge.}

[CONCERN] [SEVERITY: HIGH] {The plan says “after every code-file Write/Edit,” but the actual hook registration is under a broad `PostToolUse` matcher shared with other hooks. If the new script is added to the same group and internally filters tools, any parsing bug or matcher mismatch becomes a silent global no-op. The requirement is technically satisfied by registration, but practically broken if the hook never reaches the intended branch.}

[CONCERN] [SEVERITY: HIGH] {The advisory stdout contract creates safety theater. A hook that emits findings only as `additionalContext` is easy for an executor to ignore, easy for a later tool call to bury, and impossible to enforce. The system can claim “real-time bug/security scanning” while having no reliable operational effect on the code path that just introduced the issue.}

[CONCERN] [SEVERITY: HIGH] {The plan assumes MiniMax API availability, latency, and response shape are stable enough for a PostToolUse hook with a 10s timeout environment. At 2 AM, transient API slowness, DNS issues, rate limiting, malformed JSON, or partial responses degrade into silent exits because the hook is explicitly fail-silent. That means the system fails open exactly when reliability matters most.}

[CONCERN] [SEVERITY: HIGH] {“Fail-silent” plus “no stdout on CLEAN or skipped or error” destroys observability. In production you cannot distinguish: scanner never ran, scanner skipped by threshold, scanner skipped by extension, API failed, parsing failed, or scanner truly found nothing. The absence of output becomes meaningless, so silent failure is the default steady state.}

[CONCERN] [SEVERITY: HIGH] {The plan sends code diffs or file content to an external API immediately after every edit. That is a material data exfiltration path. The review dismisses this as “user opted in,” but opt-in does not solve overcollection: unrelated secrets in the diff, proprietary code in uncommitted changes, or accidentally edited credentials can be shipped off-box before any human notices.}

[CONCERN] [SEVERITY: HIGH] {Token logging only on “success with non-null tokens” means accounting is biased toward healthy runs and blind to failure volume. The system can show neat spend records while hiding the operational fact that half the scans are timing out, erroring, or being skipped. That satisfies logging requirements while concealing reliability collapse.}

[CONCERN] [SEVERITY: MEDIUM] {The plan depends on `computeCodexCostStrict` recognizing the exact model string used by the scanner. The project config shows `MiniMax-M2.7`, while the pricing contract references `minimax-m2.7`. A case or normalization mismatch is the simplest way for an executor to misinterpret the interface and end up with `null` cost, broken token logs, or silent omission of records.}

[CONCERN] [SEVERITY: MEDIUM] {“Current project settings.json minimax block” uses `enabled`, `model`, and task lists, but the scanner also depends on a new `scan_skip_threshold` key in `.claude/settings.json`. If the executor writes it at the wrong level, wrong file, or wrong type, the feature can default to scanning everything or nothing without obvious breakage. JSON presence is not the same as semantic correctness.}

[CONCERN] [SEVERITY: MEDIUM] {The plan assumes `appendFileSync` atomicity is enough for token-log integrity. Atomic append does not protect against malformed partial JSON lines from process crashes mid-write, duplicated records from retries, interleaving across platforms/filesystems outside the narrow Linux assumption, or unbounded growth making the log itself an operational liability.}

[CONCERN] [SEVERITY: MEDIUM] {The instructions encourage linking to `minimax-exec.js` via `require()`, but that creates a hidden runtime dependency on module format, export stability, and path correctness in a user home directory. If either hook is moved, converted to ESM, or locally modified, the scanner fails at runtime and silently exits 0 if implemented per the fail-silent rule.}

[CONCERN] [SEVERITY: MEDIUM] {The simplest executor misread is to scan file contents instead of the newly changed diff, because the plan mixes file-path triggers, git-diff fallback, and “after Write/Edit” semantics. That produces noisy findings on pre-existing code, makes the tool look low-value, and conditions users to ignore legitimate alerts.}

[CONCERN] [SEVERITY: MEDIUM] {The opposite misread is also easy: scan only the textual diff hunks and ignore surrounding context. Many security issues are context-dependent, such as taint flows, auth boundaries, import changes, dead-code activation, or semantic shifts caused by a single renamed symbol. The scanner can report “clean” on a dangerous edit because the prompt was too narrow.}

[CONCERN] [SEVERITY: MEDIUM] {PostToolUse hooks run after the write already happened. If the scan surfaces a severe issue, the code is already on disk and potentially part of subsequent automated steps. This technically meets “runs automatically after every write,” but practically it is too late to prevent propagation into tests, commits, packaging, or later agent actions.}

[CONCERN] [SEVERITY: MEDIUM] {The plan treats markdown, JSON, and planning docs as low-risk non-code, but JSON often contains policy, schema, permissions, routing, and deployment behavior. A one-line JSON edit can disable auth, widen CORS, or redirect environments. “Skip without a scan” is not a safe operational assumption.}

[CONCERN] [SEVERITY: MEDIUM] {The requirement “each successful scan’s token usage is logged” can be met even if findings are never surfaced. An executor can implement logging and silent success handling correctly while producing unusable or empty `additionalContext`. The metrics look healthy while the actual safety signal is practically broken.}

[CONCERN] [SEVERITY: LOW] {The 100-line artifact minimum incentivizes filler. An executor can satisfy the plan with boilerplate, duplicated helpers, or broad `try/catch` blocks that suppress meaningful failures. The plan measures size and string patterns, not behavioral correctness.}

[CONCERN] [SEVERITY: LOW] {The hook’s success criteria depend on string containment checks like `contains: "runMinimax"` and `pattern: "appendFileSync.*token-log"`. Those are trivial to satisfy with dead code, comments, or unreachable branches. The plan can pass review while the operational path never invokes MiniMax or never logs anything.}

[CONCERN] [SEVERITY: LOW] {At 2 AM, the most likely failure mode is operator illusion: no output appears, the edit proceeds, and everyone assumes the scanner said “clean.” In reality it may have skipped on threshold, excluded the file type, hit an API error, timed out, or crashed during parsing. The design makes the dangerous state indistinguishable from the healthy state.}

## Verdict

BLOCKED_HIGH_SEVERITY
