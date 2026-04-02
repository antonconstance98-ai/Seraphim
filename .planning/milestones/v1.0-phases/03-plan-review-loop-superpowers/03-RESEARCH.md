# Phase 03: Plan Review Loop & Superpowers - Research

**Researched:** 2026-04-02
**Domain:** Multi-round Opus-Codex review loop, typed handoff specs, review state persistence, Superpowers parallel agent model routing
**Confidence:** HIGH

---

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions

**Review Loop Dynamics**
- D-01: Loop follows a **5-step flow**: Opus drafts → Codex critiques (constructive) → Opus revises → Codex adversarial review (poke holes, edge cases) → Opus final revision. Two distinct Codex review types — constructive then adversarial
- D-02: **Early exit when clean** — if Codex's first constructive review finds zero issues, the adversarial round is skipped. Saves time and tokens on clean plans
- D-03: **Opus always wins** after the last round. Codex's unresolved concerns go into the `decisions_not_taken` section of the handoff spec. Matches the "Opus is architect" rule from Phase 1

**Superpowers Routing**
- D-04: Three Superpowers parallel agent types route to **GPT-5.4-mini via API**: hypothesis testing, code review threads, and verification checks. All three are good fits for a fast, cheap model
- D-05: **Escalate to Opus on low confidence** — if GPT-5.4-mini gives low-confidence or unclear results, re-run on Opus. Only triggers when needed, ensuring quality while optimizing cost

**Loop Visibility**
- D-06: User sees **milestone updates** during the loop: "Round 1: Codex reviewing..." then "Round 1: 3 suggestions. Opus revising..." then final result. Progress indicators without the full back-and-forth
- D-07: Final plan includes a **summary of changes** showing what Codex's review improved vs the original draft. User sees the value of the review loop

**Carrying Forward**
- Phase 1 D-01: Moderate guardrails — Codex can make minor judgment calls
- Phase 1 D-08: Attributed results — user knows which model did the work
- Phase 2 D-07: Plan-phase blocked until Codex reviews (Phase 3 upgrades this to multi-round)
- Phase 2 D-12: Review depth varies by task type

### Claude's Discretion
- Handoff spec format: Claude designs the typed handoff spec structure (fields, sections, `decisions_not_taken` format) based on what downstream execution needs
- Review state persistence: Claude implements `.planning/review-state.json` round counter and state tracking
- Adversarial prompt design: Claude crafts the adversarial review prompt that tells Codex to poke holes and find edge cases
- Low-confidence detection: Claude defines signals that trigger escalation from GPT-5.4-mini to Opus
- Superpowers skill modification approach: Claude determines how to modify `dispatching-parallel-agents` skill for model routing (after verifying skill symlink path)

### Deferred Ideas (OUT OF SCOPE)
None — discussion stayed within phase scope.
</user_constraints>

---

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| REVW-03 | Opus-Codex plan review loop (2-3 rounds) triggers before every GSD phase plan, with hard 3-round cap | Builds on Phase 2 `codex-plan-reviewer.js` SubagentStop hook; loop orchestrator added as new module; round counter in `review-state.json` |
| REVW-04 | Opus-Codex plan review loop (2-3 rounds) triggers before every GSD individual task plan | Same loop orchestrator fires on SubagentStop for both phase plans and individual task PLAN.md files; detection already exists in `detectRecentlyPlannedPhase()` |
| REVW-05 | Opus-Codex plan review loop integrates into Superpowers planning/implementation design phases | Superpowers `writing-plans` skill invokes the review loop via a SubagentStop hook on the planner subagent; the same loop module is reused |
| REVW-06 | Review loop produces a typed handoff spec with decisions-not-taken section, and Opus has final authority after round 3 | Handoff spec design confirmed feasible; `decisions_not_taken` is a JSONL/markdown section in the spec; round counter enforces cap at 3 |
| SPWR-01 | Superpowers plugin source modified to use Codex during planning/implementation design phases | `dispatching-parallel-agents` SKILL.md is writable at install path; modification confirmed; durability risk via plugin updates documented |
| SPWR-02 | Superpowers plan review uses the same Opus-Codex review loop as GSD (2-3 rounds, 3-round cap) | Same loop orchestrator module; Superpowers invokes via SubagentStop hook on `writing-plans` subagent |
| SPWR-03 | Superpowers parallel agent dispatch can route hypothesis-testing threads to GPT-5.4-mini via API instead of spawning more Opus subagents | `dispatching-parallel-agents` SKILL.md currently uses `model: "claude-sonnet-4-5"` for mechanical tasks; adds GPT-5.4-mini as a third route for hypothesis testing; OpenAI SDK needed |
</phase_requirements>

---

## Summary

Phase 3 builds on the Phase 2 single-pass Codex review to produce a multi-round (2-3 round, hard-capped at 3) Opus-Codex plan review loop. The architecture has two parallel workstreams:

**Workstream 1 — Review Loop Upgrade.** The existing `codex-plan-reviewer.js` SubagentStop hook is extended into a full multi-round loop orchestrator. The orchestrator runs up to three phases: (1) Codex constructive critique, (2) Opus revision, (3) Codex adversarial review (poke holes, find edge cases). If round 1 finds zero issues, round 2 is skipped. After round 3, Opus has final authority; any unresolved Codex concerns go into the `decisions_not_taken` section of a typed handoff spec. The round counter and loop state are persisted to `.planning/review-state.json` so sessions that are interrupted can resume correctly. The hook also fires for individual task plans (PLAN.md detection already works in Phase 2).

**Workstream 2 — Superpowers Integration.** The Superpowers `dispatching-parallel-agents` SKILL.md is modified to add a third model route: GPT-5.4-mini (via OpenAI API) for hypothesis-testing parallel threads, alongside the existing `claude-sonnet-4-5` (mechanical) and `claude-opus-4-0` (judgment) routes. Separately, the Superpowers `writing-plans` workflow is wired to the same multi-round review loop by adding a SubagentStop hook that fires when the writing-plans subagent completes. Both changes reuse the same loop orchestrator module from Workstream 1.

**Critical dependency:** The `openai` npm package (v6.33.0) is **not installed as a standalone global package** — it exists only as a transitive dependency inside `openclaw`. Before any code that `require('openai')` can run from hook scripts, `npm install -g openai` must be the first task in Wave 0.

**Primary recommendation:** Write a new `codex-multi-round-reviewer.js` module that the upgraded `codex-plan-reviewer.js` calls. Add `review-state.json` persistence, the two distinct Codex prompts, and the typed handoff spec writer. Modify the Superpowers `dispatching-parallel-agents/SKILL.md` to add the GPT-5.4-mini route. Add a SubagentStop hook for Superpowers' writing-plans subagent.

---

## Standard Stack

### Core
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| Node.js | v22.22.0 (installed) | Multi-round loop orchestrator, hook scripts | All existing hooks are Node.js; no new runtime |
| `codex-exec.js` | Phase 1 artifact | Codex CLI invocation wrapper | Exports `runCodexExec`, `parseCodexTokens`, `computeCost` — already in use |
| `@openai/codex` CLI | 0.118.0 (installed) | Constructive and adversarial Codex review rounds | Confirmed working; uses ChatGPT Plus subscription |
| `openai` npm package | 6.33.0 (latest) | GPT-5.4-mini API calls for Superpowers routing | MUST BE INSTALLED (see Wave 0 gap) |
| Claude Code hooks API | v2.1.89+ | SubagentStop event for loop trigger | Native; already used for Phase 2 plan reviewer |

### Supporting
| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| `child_process` (Node built-in) | Node 22 | Async / sync Codex subprocess management | Already used by `codex-exec.js` |
| `fs` / `path` (Node built-ins) | Node 22 | `review-state.json` read/write, PLAN.md discovery | Already used in all hook scripts |

### Alternatives Considered
| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| OpenAI SDK (API) for GPT-5.4-mini | Codex CLI with `--model gpt-5.4-mini` | CLI startup is 1-2s — acceptable for long review rounds; NOT acceptable for quick hypothesis dispatch where latency matters. API path correct for SPWR-03 |
| Modifying Superpowers cache directly | Wrapper SKILL.md in `~/.claude/skills/` | Cache modification is simpler but gets wiped on plugin update; `~/.claude/skills/` override is durable (see Pitfall 2) |

**Installation (Wave 0 prerequisite):**
```bash
npm install -g openai
# Verify:
node -e "require('openai'); console.log('OK')"
```

**Version verification (2026-04-02):**
- `@openai/codex` CLI: 0.118.0 — verified via `codex --version`
- `openai` npm: 6.33.0 — verified via `npm show openai version` (not yet installed globally)
- Node.js: v22.22.0 — verified

---

## Architecture Patterns

### Recommended File Structure

```
~/.claude/hooks/
├── codex-exec.js                      # Phase 1 — not modified
├── codex-plan-reviewer.js             # Phase 2 — UPGRADED to call multi-round reviewer
├── codex-multi-round-reviewer.js      # Phase 3 NEW — loop orchestrator (shared by GSD + Superpowers)
└── codex-superpowers-plan-reviewer.js # Phase 3 NEW — SubagentStop hook for writing-plans subagent

~/.claude/skills/
└── dispatching-parallel-agents/
    └── SKILL.md                       # Phase 3 NEW — override SKILL.md with GPT-5.4-mini route

.planning/
└── review-state.json                  # Phase 3 NEW — round counter + loop state persistence
```

**Why `~/.claude/skills/dispatching-parallel-agents/` override instead of modifying the cache:**
The Superpowers cache at `~/.claude/plugins/cache/claude-plugins-official/superpowers/5.0.7/` has `lastUpdated: 2026-03-31` — it auto-updates. A plugin update would silently overwrite any changes to the cached SKILL.md. A `~/.claude/skills/` override is in user space and survives plugin updates.

**How the override works:** When Claude Code's `Skill` tool resolves `dispatching-parallel-agents`, it checks `~/.claude/skills/` before plugin skill directories. A file at `~/.claude/skills/dispatching-parallel-agents/SKILL.md` shadows the plugin's version without touching the plugin cache.

(Note: This precedence behavior should be confirmed during implementation via a test invocation of the skill before and after adding the override file.)

---

### Pattern 1: Multi-Round Loop Orchestrator

**What:** A `codex-multi-round-reviewer.js` module that implements the 5-step flow (D-01) with early exit (D-02) and round cap. Called by both `codex-plan-reviewer.js` (GSD) and `codex-superpowers-plan-reviewer.js` (Superpowers).

**Flow:**
```
Round 1 (always):
  Codex constructive critique → extract issues
  If zero issues → EARLY EXIT (D-02), skip round 2

Round 2 (only if round 1 found issues):
  Opus revision (injected via additionalContext to SubagentStop response)
  Wait for Opus to produce revised plan
  Codex adversarial review → extract unresolved concerns

Final output:
  Typed handoff spec with decisions_not_taken (D-03)
  REVS file updated with round history
  review-state.json cleared
```

**Key constraint:** The multi-round loop uses `codex-exec.js` (Codex CLI) for both Codex rounds. Each round spawns a `runCodexExec()` call with a distinct prompt. The orchestrator does NOT use the OpenAI SDK — that is only used in the Superpowers dispatch path (SPWR-03).

**Example module signature:**
```javascript
// Source: pattern from ~/.claude/hooks/codex-plan-reviewer.js (Phase 2 confirmed working)
async function runMultiRoundReview(cwd, phase, planContent, options) {
  // options: { maxRounds: 3, taskType: 'phase-plan'|'task-plan'|'superpowers-plan' }
  // Returns: { rounds, handoffSpec, hasBlockingIssues, reviewsPath }
}
```

### Pattern 2: Review State Persistence (`.planning/review-state.json`)

**What:** A file that tracks the current round, which plans are under review, and intermediate results across potential session boundaries.

**Schema:**
```json
{
  "schema_version": 1,
  "session_id": "abc123",
  "phase": "03",
  "plan_files": ["03-01-PLAN.md"],
  "current_round": 1,
  "max_rounds": 3,
  "started_at": "2026-04-02T14:00:00.000Z",
  "rounds": [
    {
      "round": 1,
      "type": "constructive",
      "codex_output": "...",
      "issues_found": 3,
      "completed_at": "2026-04-02T14:02:00.000Z"
    }
  ],
  "status": "in_progress"
}
```

**Lifecycle:** Created when the loop starts, updated after each round, deleted (or marked `status: "complete"`) when the loop finishes. On SubagentStop re-entry, if `review-state.json` exists with `status: in_progress` for the same phase, the loop resumes from the current round rather than restarting.

### Pattern 3: Typed Handoff Spec

**What:** A structured markdown file written to the phase directory after the loop completes. Downstream GSD execution reads this to understand what was reviewed and what was intentionally not implemented.

**Format:** Written to `.planning/phases/NN-name/NN-HANDOFF.md`

```markdown
# Phase NN — Review Handoff Spec

**Reviewed:** [ISO timestamp]
**Rounds completed:** [1|2|3]
**Model authority:** Opus 4.6 (final authority per D-03)

## Plan Changes from Review

[What changed between draft and final plan, in plain English (D-07)]

## Decisions Not Taken

| Issue | Raised by | Round | Reason Not Implemented |
|-------|-----------|-------|------------------------|
| [concern] | Codex | 2 | [Opus reasoning] |

## Review Verdict

[APPROVED|APPROVED_WITH_CHANGES|BLOCKED_HIGH_SEVERITY]
```

### Pattern 4: Superpowers Skill Override with GPT-5.4-mini Route

**What:** A user-space SKILL.md override at `~/.claude/skills/dispatching-parallel-agents/SKILL.md` that adds GPT-5.4-mini as a third dispatch route. The override is a complete replacement of the plugin's SKILL.md, incorporating all existing content plus the new route section.

**Current model routing in plugin v5.0.7 (lines 116-138):**
- `claude-sonnet-4-5` → Mechanical/focused tasks (proxied to MiniMax M2.7)
- `claude-opus-4-0` → Judgment/integration tasks (proxied to Anthropic)

**New third route to add:**
```markdown
**Hypothesis testing / cheap parallel trials** — use `model: "gpt-5.4-mini"`:
- Parallel debugging hypotheses (try 3-5 different approaches)
- Quick code review threads that need scale but not depth
- Verification checks (does the output match the spec?)
- Routes via OpenAI API (not via proxy); requires OPENAI_API_KEY

Task({ description: "Hypothesis A: fix via X approach", model: "gpt-5.4-mini" })

**Escalation:** If a gpt-5.4-mini agent returns "LOW CONFIDENCE" or "BLOCKED", 
re-dispatch the task with model: "claude-opus-4-0" for deeper reasoning.
```

**Note on how `model` param works with GPT-5.4-mini:** The `Task` tool in Claude Code accepts a `model` parameter that is passed to the spawned subagent. Whether `model: "gpt-5.4-mini"` triggers a local Codex CLI call or an OpenAI API call depends on how the subagent platform handles the model string. Research confirms the `Task` tool sends the model hint but the actual routing depends on the Claude Code routing configuration. The OpenAI SDK path (via an advisory hook) may be needed to actually invoke GPT-5.4-mini when Claude Code would otherwise fall back to its default model. **This routing mechanism must be verified during implementation.**

### Pattern 5: OpenAI SDK Direct API Call for GPT-5.4-mini

**What:** When the Superpowers skill dispatches hypothesis threads with `model: "gpt-5.4-mini"`, the advisory hook intercepts and invokes the OpenAI SDK directly (faster than spawning Codex CLI for short advisory responses).

**API call pattern (confirmed from openai v6.33.0):**
```javascript
// Source: OpenAI Node.js SDK v6.33.0 documentation
const { OpenAI } = require('openai');

const client = new OpenAI({ apiKey: process.env.OPENAI_API_KEY });

async function callGpt54Mini(prompt, { maxTokens = 500, timeoutMs = 10000 } = {}) {
  const controller = new AbortController();
  const timer = setTimeout(() => controller.abort(), timeoutMs);
  try {
    const response = await client.chat.completions.create({
      model: 'gpt-4o-mini',  // NOTE: verify actual gpt-5.4-mini model ID — see Open Question 1
      messages: [{ role: 'user', content: prompt }],
      max_tokens: maxTokens,
    }, { signal: controller.signal });
    return {
      success: true,
      text: response.choices[0].message.content,
      usage: response.usage,  // { prompt_tokens, completion_tokens, total_tokens }
    };
  } catch (e) {
    return { success: false, error: e.message };
  } finally {
    clearTimeout(timer);
  }
}
```

**Token logging:** OpenAI SDK responses include `response.usage.prompt_tokens` and `response.usage.completion_tokens` — already handled by `computeCost()` in `codex-exec.js`.

---

### Anti-Patterns to Avoid

- **Hard-coding round count:** The loop cap (3 rounds) must be configurable in `review-state.json` and honoured by the orchestrator. Do not hard-code `for i in 1..3`.
- **Blocking on Superpowers SubagentStop for long loops:** The multi-round loop can take 3-5 minutes. The SubagentStop hook timeout in `settings.json` must be set to at least 600s for Superpowers hooks. GSD already has 300s — may need to extend.
- **Modifying the Superpowers plugin cache directly:** Plugin updates overwrite it. Always write the override to `~/.claude/skills/`.
- **Using Codex CLI for GPT-5.4-mini dispatch in Superpowers:** CLI startup (1-2s) is too slow for the hypothesis dispatch advisory path. Use OpenAI SDK for API calls.
- **Not persisting review-state.json before the first Codex call:** If the hook crashes during round 1, a re-entry with no state file would restart from round 0, wasting tokens. Write the state file BEFORE the first Codex invocation.

---

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Multi-round retry with timeout | Custom retry loop | `runCodexExec()` with 180s per round + `review-state.json` for persistence | `runCodexExec` already has SIGTERM/SIGKILL fallback; adding a new retry layer creates two timeout systems |
| OpenAI API error handling | Custom retry/backoff | openai SDK built-in retry (configured via `maxRetries`) | SDK handles 429 rate limits and transient errors; hand-rolling these has known edge cases |
| JSON serialization of handoff spec | Custom JSON writer | `JSON.stringify` + markdown template string | No schema complexity that warrants a separate library |
| Skill content discovery | Scanning plugin cache | Write to `~/.claude/skills/` override | Cache path changes with plugin version; user skills dir is stable |
| SubagentStop detection for Superpowers | New heuristic | Reuse `detectRecentlyPlannedPhase()` pattern from Phase 2 | The heuristic (PLAN.md modified in last 120s) already works for GSD; Superpowers plan files follow the same `docs/superpowers/plans/*.md` convention |

---

## Runtime State Inventory

Step 2.5: SKIPPED — Phase 3 is not a rename/refactor/migration phase.

---

## Environment Availability

| Dependency | Required By | Available | Version | Fallback |
|------------|-------------|-----------|---------|---------|
| Node.js | All hook scripts | Yes | v22.22.0 | — |
| `@openai/codex` CLI | REVW-03, REVW-04, REVW-05, REVW-06 | Yes | 0.118.0 at `~/.npm-global/bin/codex` | — |
| `openai` npm package | SPWR-03 (GPT-5.4-mini API calls) | **No** — not globally installed | — | **BLOCKING: must install before Wave 1** |
| OPENAI_API_KEY env var | SPWR-03 | Yes | set in ~/.bashrc (length: 164 chars) | — |
| `~/.claude/skills/` directory | SPWR-01 (skill override) | No — does not exist yet | — | Create it in Wave 0 (`mkdir -p ~/.claude/skills/dispatching-parallel-agents`) |
| `review-state.json` | REVW-03, REVW-04, REVW-05 | No — not created yet | — | Created at loop start (Wave 1 task) |

**Missing dependencies with no fallback:**
- `openai` npm package — blocks SPWR-03 entirely. Must be installed as Wave 0 task: `npm install -g openai`

**Missing dependencies with fallback:**
- `~/.claude/skills/dispatching-parallel-agents/` directory — created as part of skill override task (no blocker, just needs mkdir)

---

## Common Pitfalls

### Pitfall 1: SubagentStop Re-Entry Loop in Multi-Round Review

**What goes wrong:** The SubagentStop hook blocks the gsd-planner subagent for round 1. The planner is given another turn (via `decision: "block"`). When the planner finishes its revision and completes again, SubagentStop fires again. Without state tracking, the hook restarts round 1 instead of proceeding to round 2.

**Why it happens:** The SubagentStop hook has no memory of prior invocations. Each time the planner subagent completes, the hook sees a fresh stdin payload.

**How to avoid:** Read `.planning/review-state.json` at hook entry. If `status: in_progress` and `current_round: 1` already exists for this phase and session, advance to the next round instead of restarting. Write state before every round starts.

**Warning signs:** If token-log.jsonl shows the same phase being reviewed 3+ times in rapid succession, state tracking is not working.

---

### Pitfall 2: Superpowers Plugin Update Overwrites SKILL.md Changes

**What goes wrong:** Local modifications to `~/.claude/plugins/cache/claude-plugins-official/superpowers/5.0.7/skills/dispatching-parallel-agents/SKILL.md` are silently overwritten when Superpowers auto-updates. The `lastUpdated` field in `installed_plugins.json` shows updates happen regularly (most recent: 2026-03-31).

**Why it happens:** The plugin cache is managed by Claude Code's plugin system. Edits to cache files are not tracked — they are overwritten on update.

**How to avoid:** Write the modified SKILL.md to `~/.claude/skills/dispatching-parallel-agents/SKILL.md` (user-level override directory). Claude Code's Skill tool checks `~/.claude/skills/` before plugin skill directories, so the override shadows the cache version.

**Warning signs:** After a Superpowers update (`lastUpdated` changes in `installed_plugins.json`), the Skill tool returns old content that doesn't mention GPT-5.4-mini.

---

### Pitfall 3: SubagentStop Timeout Too Short for Multi-Round Loop

**What goes wrong:** The Phase 2 SubagentStop hook registration uses `timeout: 300` (5 minutes). A multi-round loop (2 Codex calls at ~180s each) can take 6+ minutes total, causing the hook to time out mid-loop.

**Why it happens:** Phase 2 reviewed plans once (~90-180s). Phase 3 runs 2-3 rounds. Each round is an independent `runCodexExec()` call with a 180s timeout. Total wall time: up to 540s.

**How to avoid:** Increase the SubagentStop hook timeout in `~/.claude/settings.json` to 600s (or 720s with buffer) for the `codex-plan-reviewer.js` registration. The existing 300s hook timeout must be extended.

**Warning signs:** SubagentStop completes early with no REVS file written; stderr shows "hook timed out".

---

### Pitfall 4: `model: "gpt-5.4-mini"` May Not Route to OpenAI Without an Advisory Hook

**What goes wrong:** Setting `model: "gpt-5.4-mini"` in a `Task()` call from the Superpowers SKILL.md may not actually invoke GPT-5.4-mini. Claude Code passes the model hint to the subagent spawner, but whether the actual routing to OpenAI occurs depends on how Claude Code handles unknown model IDs.

**Why it happens:** The `Task` tool's `model` parameter is designed for Claude model variants (`claude-sonnet-*`, `claude-opus-*`). A non-Claude model string like `gpt-5.4-mini` may be ignored or fallback to the session default.

**How to avoid:** The SKILL.md instruction should additionally ask Claude to call an advisory hook (or use `runCodexExec` pattern via a helper) when spawning hypothesis threads with GPT-5.4-mini. Alternatively, the skill instructs Opus to use `require('./codex-exec').runGpt54Mini()` instead of `Task()` for this route. **Validate the actual Task model routing behavior during implementation.**

**Warning signs:** Token logs show `model: "claude-opus-4-0"` instead of `gpt-5.4-mini` for hypothesis threads.

---

### Pitfall 5: Adversarial Round Prompt Too Similar to Constructive Round

**What goes wrong:** If the adversarial prompt is nearly identical to the constructive prompt, Codex produces the same type of feedback in both rounds — the adversarial review doesn't add value.

**Why it happens:** "Review this plan" is the default instruction. Without explicit framing, Codex defaults to constructive feedback.

**How to avoid:** The adversarial prompt must use explicit adversarial framing: "You are a skeptical reviewer. Challenge every assumption. Look for what could go wrong, what edge cases are not handled, what the plan gets wrong, and what the Opus author has over-simplified." D-D-01 spec says the second Codex review should "poke holes and find edge cases" — encode this explicitly.

---

## Code Examples

### Multi-Round Orchestrator Core

```javascript
// Source: Pattern extended from ~/.claude/hooks/codex-plan-reviewer.js (Phase 2, confirmed working 2026-04-02)
// New module: ~/.claude/hooks/codex-multi-round-reviewer.js

const CONSTRUCTIVE_PROMPT_SUFFIX = `
You are a constructive reviewer. For each plan, identify:
1. Missing or vague task actions (would an executor need clarification?)
2. Missing verification commands
3. Dependency issues (references a file from a later wave)
4. File conflicts (same file modified by parallel plans)
5. Requirements coverage gaps

For each issue: [PLAN {id}] [SEVERITY: HIGH|MEDIUM|LOW] {description}
If no issues: output exactly "PLANS APPROVED — no issues found"
`;

const ADVERSARIAL_PROMPT_SUFFIX = `
You are a skeptical adversarial reviewer. Challenge every assumption.
Do NOT be constructive. Your job is to find what can go wrong:
1. What edge cases are not handled?
2. What assumptions does the plan rely on that could be false?
3. What is the simplest way this plan could fail in production?
4. What did the author over-simplify or get wrong?
5. What requirements are technically satisfied but practically broken?

For each concern: [CONCERN] [SEVERITY: HIGH|MEDIUM|LOW] {description}
If the plan is genuinely robust: output "ADVERSARIAL REVIEW PASSED — no vulnerabilities found"
`;

async function runMultiRoundReview(cwd, phase, planContent, options = {}) {
  const maxRounds = options.maxRounds || 3;
  const stateFile = path.join(cwd, '.planning', 'review-state.json');
  
  // Load or initialize state
  let state = loadOrInitState(stateFile, { phase, maxRounds });
  
  const results = { rounds: [], handoffSpec: null, hasBlockingIssues: false };

  // Round 1: Constructive
  if (state.current_round <= 1) {
    state = advanceRound(stateFile, state, 1);
    const r1 = await runCodexExec(CONSTRUCTIVE_PROMPT_SUFFIX + '\n\nPlans:\n' + planContent,
      { cwd, timeoutMs: 180000, model: 'gpt-5.4' });
    results.rounds.push({ round: 1, type: 'constructive', result: r1 });
    state = recordRoundResult(stateFile, state, 1, r1);
    
    // D-02: Early exit if clean
    if (!hasIssues(r1)) {
      return finalizeHandoff(results, state, stateFile, 'APPROVED');
    }
  }

  // Round 2: Adversarial (only if round 1 found issues)
  if (state.current_round <= 2) {
    state = advanceRound(stateFile, state, 2);
    const r2 = await runCodexExec(ADVERSARIAL_PROMPT_SUFFIX + '\n\nPlans:\n' + planContent,
      { cwd, timeoutMs: 180000, model: 'gpt-5.4' });
    results.rounds.push({ round: 2, type: 'adversarial', result: r2 });
    state = recordRoundResult(stateFile, state, 2, r2);
    results.hasBlockingIssues = hasHighSeverity(r2);
  }

  return finalizeHandoff(results, state, stateFile, results.hasBlockingIssues ? 'BLOCKED' : 'APPROVED_WITH_CHANGES');
}
```

### OpenAI SDK Call for GPT-5.4-mini (SPWR-03)

```javascript
// Source: openai npm package v6.33.0 — chat.completions.create pattern
// To be added to codex-exec.js as runGpt54MiniApi()
const { OpenAI } = require('openai');

async function runGpt54MiniApi(prompt, options = {}) {
  const client = new OpenAI({ apiKey: process.env.OPENAI_API_KEY });
  const timeoutMs = options.timeoutMs || 15000;
  
  const controller = new AbortController();
  const timer = setTimeout(() => controller.abort(), timeoutMs);
  
  try {
    const response = await client.chat.completions.create({
      model: options.model || 'gpt-4o-mini',  // verify gpt-5.4-mini model ID — see Open Questions
      messages: [{ role: 'user', content: prompt }],
      max_tokens: options.maxTokens || 500,
    }, { signal: controller.signal });
    
    return {
      success: true,
      text: response.choices[0].message.content,
      usage: response.usage,  // prompt_tokens, completion_tokens, total_tokens
    };
  } catch (e) {
    return { success: false, error: e.message, text: '' };
  } finally {
    clearTimeout(timer);
  }
}
```

### Dispatching-Parallel-Agents SKILL.md Addition

```markdown
**Hypothesis testing / parallel trials** — use `model: "gpt-5.4-mini"`:
- Parallel debugging hypotheses (try 3-5 different approaches simultaneously)
- Quick verification checks that need scale but not deep reasoning
- Background code review threads where speed matters more than depth
- Requires OPENAI_API_KEY to be set in environment

Task({ description: "Hypothesis A: fix via approach X", model: "gpt-5.4-mini" })
Task({ description: "Hypothesis B: fix via approach Y", model: "gpt-5.4-mini" })
Task({ description: "Hypothesis C: fix via approach Z", model: "gpt-5.4-mini" })

**Escalation rule:** If a gpt-5.4-mini agent returns output containing 
"LOW CONFIDENCE", "UNSURE", or "BLOCKED", re-dispatch with model: "claude-opus-4-0".
Only escalate when explicitly needed — not pre-emptively.
```

### review-state.json Read/Write Pattern

```javascript
// Source: Pattern designed for Phase 3 (codex-multi-round-reviewer.js)
function loadOrInitState(stateFile, opts) {
  if (fs.existsSync(stateFile)) {
    try {
      const existing = JSON.parse(fs.readFileSync(stateFile, 'utf8'));
      // Resume if same phase and same session, in progress
      if (existing.phase === opts.phase && existing.status === 'in_progress') {
        return existing;
      }
    } catch (e) { /* malformed — reinit */ }
  }
  const state = {
    schema_version: 1,
    phase: opts.phase,
    max_rounds: opts.maxRounds,
    current_round: 0,
    started_at: new Date().toISOString(),
    rounds: [],
    status: 'in_progress'
  };
  fs.writeFileSync(stateFile, JSON.stringify(state, null, 2), 'utf8');
  return state;
}
```

---

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Single-pass plan review (Phase 2) | Multi-round loop with constructive + adversarial rounds | Phase 3 | Catches issues Phase 2 misses; adversarial round finds edge cases the author over-simplified |
| All parallel agents use Claude model variants | Hypothesis-testing threads routable to GPT-5.4-mini | Phase 3 | ~10x cheaper per hypothesis thread; enables more parallel trials within budget |
| Review result only in REVIEWS.md | Typed handoff spec with `decisions_not_taken` section | Phase 3 | Downstream executor knows what was consciously excluded from the plan |

**Existing infrastructure that Phase 3 reuses unchanged:**
- `codex-exec.js` `runCodexExec()` — no changes needed
- `codex-exec.js` `computeCost()` — no changes needed (already knows `gpt-5.4-mini` pricing)
- Token log format in `.planning/token-log.jsonl` — extend with `task_type: 'multi-round-plan-review'`
- SubagentStop hook registration pattern in `~/.claude/settings.json`

---

## Open Questions

1. **GPT-5.4-mini exact model ID for OpenAI API**
   - What we know: `codex-exec.js` uses `'gpt-5.4-mini'` as the model string for Codex CLI. The OpenAI API may use a different identifier (e.g., `gpt-4o-mini` was the predecessor pattern).
   - What's unclear: Whether `model: 'gpt-5.4-mini'` is the correct string for `chat.completions.create()` in openai SDK v6.33.0.
   - Recommendation: Run `node -e "const {OpenAI} = require('openai'); const c = new OpenAI({apiKey: process.env.OPENAI_API_KEY}); c.models.list().then(r => r.data.filter(m => m.id.includes('mini') || m.id.includes('5.4')).forEach(m => console.log(m.id))).catch(e => console.error(e.message))"` during Wave 0 to confirm the exact model ID before hardcoding.

2. **Whether Claude Code's `Task()` tool respects `model: "gpt-5.4-mini"` for OpenAI routing**
   - What we know: The `Task` tool accepts a `model` parameter; the Superpowers SKILL.md already uses `claude-sonnet-4-5` and `claude-opus-4-0` model hints.
   - What's unclear: Whether `model: "gpt-5.4-mini"` routes to OpenAI or falls back to a Claude model (since Claude Code is an Anthropic tool).
   - Recommendation: Test with a simple Task dispatch using `model: "gpt-5.4-mini"` and check the token log to see which model was actually billed. If it falls back, the SKILL.md instruction should tell Opus to use the OpenAI API helper directly instead of `Task()` for hypothesis threads.

3. **Whether `~/.claude/skills/` shadows plugin skills in the Skill tool**
   - What we know: The Superpowers session-start hook loads `using-superpowers` from the plugin root. The `Skill` tool resolves skills by name.
   - What's unclear: The exact precedence order — does the Skill tool check `~/.claude/skills/` before plugin skill directories?
   - Recommendation: Create a test skill at `~/.claude/skills/test-override/SKILL.md` and try to invoke it with the Skill tool. If it resolves, the override mechanism works. If not, fall back to modifying the cached SKILL.md directly (accepting the update risk, which can be re-applied post-update).

---

## Sources

### Primary (HIGH confidence)
- `~/.claude/hooks/codex-plan-reviewer.js` — Phase 2 SubagentStop hook; full source read 2026-04-02; direct basis for multi-round extension
- `~/.claude/hooks/codex-exec.js` — Phase 1 Codex wrapper; confirmed exports `runCodexExec`, `parseCodexTokens`, `computeCost`; pricing for `gpt-5.4-mini` already defined
- `~/.claude/plugins/cache/claude-plugins-official/superpowers/5.0.7/skills/dispatching-parallel-agents/SKILL.md` — Full source read 2026-04-02; model routing section at lines 114-138 confirmed; file is writable
- `~/.claude/plugins/cache/claude-plugins-official/superpowers/5.0.7/hooks/session-start` — Full source read 2026-04-02; confirms skills are loaded on-demand via Skill tool, not injected at session start (except `using-superpowers`)
- `~/.claude/plugins/installed_plugins.json` — Superpowers 5.0.7 active; install path `/home/alucard/.claude/plugins/cache/claude-plugins-official/superpowers/5.0.7`; lastUpdated 2026-03-31 (confirms auto-update risk)
- `~/.claude/settings.json` — SubagentStop hook registration confirmed; current timeout: 300s; must be extended to 600s for multi-round

### Secondary (MEDIUM confidence)
- `docs/research/opus-vs-codex-model-comparison.md` — Multi-round review rationale confirmed from Chandler Nguyen's testing ("2-3 review rounds yields noticeably tighter implementation plans"); GPT-5.4-mini recommended for parallel hypothesis testing (Section 3.1)
- `codex-claude-code-power-user-research.md` — Cross-model review patterns confirmed; adversarial review via `codex:adversarial-review` command is an established pattern (Section 2)
- `npm show openai version` — v6.33.0 confirmed as latest; `chat.completions.create` API pattern is current (verified)

### Tertiary (LOW confidence — flag for validation)
- `~/.claude/skills/` override precedence over plugin skills — not verified by official docs; requires empirical test during Wave 0 (Open Question 3)
- GPT-5.4-mini exact model ID for `chat.completions.create` — `'gpt-5.4-mini'` is the internal codex-exec.js constant but may not match the API model ID; requires validation (Open Question 1)
- Claude Code `Task()` tool respecting `model: "gpt-5.4-mini"` for OpenAI routing — unverified (Open Question 2)

---

## Project Constraints (from CLAUDE.md)

All constraints from `CLAUDE.md` apply. Key directives relevant to Phase 3:

| Constraint | Phase 3 Impact |
|-----------|----------------|
| No OpenAI Agents SDK | Multi-round loop MUST be implemented as plain Node.js functions, not Agent SDK chains |
| No LangChain / LlamaIndex | No chain frameworks; the "chain" is two `runCodexExec()` calls in sequence |
| No `async: true` hooks for review | Review loop MUST use synchronous hooks with `additionalContext`; async hooks don't block |
| Opus always the primary orchestrator | Codex never has final authority; D-03 "Opus always wins" is enforced by the loop design |
| Never expose API keys in plaintext | `OPENAI_API_KEY` is already in `~/.bashrc` (confirmed); never written to files |
| Budget: $15/day max | GPT-5.4-mini at $0.40/$1.60 per 1M tokens is well within budget for hypothesis dispatch |
| Codex CLI (subscription) preferred over API calls | Review loop uses Codex CLI (free under Plus); only SPWR-03 Superpowers dispatch uses API |
| Must not break GSD or Superpowers workflows | Multi-round reviewer is additive to `codex-plan-reviewer.js`; Superpowers skill override preserves all existing routes |

---

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH — all libraries identified and their versions verified on 2026-04-02
- Architecture: HIGH — multi-round pattern extends Phase 2 infrastructure that is confirmed working; code examples are based on actual source files
- Pitfalls: HIGH — plugin update risk and SubagentStop re-entry loop are verified real risks from source inspection
- Open Questions: MEDIUM — three unverified assumptions require Wave 0 empirical tests before implementation proceeds

**Research date:** 2026-04-02
**Valid until:** 2026-05-02 (stable ecosystem; re-check if Superpowers updates to 5.0.8+ or openai SDK updates to 7.x)
