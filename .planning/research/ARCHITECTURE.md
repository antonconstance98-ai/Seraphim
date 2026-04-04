# Architecture Research

**Domain:** ML-driven self-optimization layer for a hook-based multi-model CLI orchestration system
**Researched:** 2026-04-03
**Confidence:** HIGH — based on live codebase inspection of all 18 hooks and settings.json

---

## System Overview

The existing system is a synchronous hook pipeline where each hook receives JSON on stdin, does
its work, and writes JSON to stdout. The ML self-optimization layer adds three new concerns
without modifying the existing pipeline: decision capture, offline analysis, and config propagation.

```
┌────────────────────────────────────────────────────────────────────────────────┐
│                            Claude Code Session                                  │
│                                                                                 │
│  PreToolUse chain              PostToolUse chain          Stop / SubagentStop   │
│  ┌──────────────────┐          ┌──────────────────┐       ┌──────────────────┐  │
│  │ settings-guard   │          │ gsd-context-mon  │       │ codex-review-    │  │
│  │ gsd-prompt-guard │          │ codex-token-     │       │ gate  (Stop)     │  │
│  │ codex-router     │          │   logger         │       │ codex-plan-      │  │
│  └────────┬─────────┘          │ codex-wave-      │       │   reviewer (Sub) │  │
│           │                    │   validator      │       │ codex-superpow-  │  │
│   reads   │                    │ minimax-post-    │       │   ers-reviewer   │  │
│  config   │                    │   scan           │       └────────┬─────────┘  │
│           │                    │ minimax-compress │                │             │
│           │                    ├──────────────────┤                │             │
│           │                    │ decision-logger  │◀───────────────┘             │
│           │                    │  (NEW — last)    │  also runs as Stop advisory  │
│           │                    └────────┬─────────┘                              │
└───────────┼──────────────────────────── │ ───────────────────────────────────────┘
            │                             │
            │                             ▼
            │              ┌──────────────────────────────┐
            │              │  decision-log.jsonl           │
            │              │  (per-project + global)       │
            │              └──────────────┬───────────────┘
            │                             │
            │                    [SessionStart]
            │                             │
            │                             ▼
            │              ┌──────────────────────────────┐
            │              │  ml-analyzer.js  (NEW)        │
            │              │  reads decision-log.jsonl     │
            │              │  runs weighted statistics     │
            │              │  writes recommendations.json  │
            │              │  calls config-writer.js if    │
            │              │  confidence >= threshold      │
            │              └──────────────┬───────────────┘
            │                             │
            │                             ▼
            │              ┌──────────────────────────────┐
            │              │  config-writer.js  (NEW)      │
            │              │  atomic write-then-rename     │
            │◀─────────────│  updates .claude/settings.    │
            │              │  json or .planning/config.    │
     behavior              │  json                        │
     changes               │  appends adjustment-log.jsonl │
     next call             └──────────────────────────────┘
```

---

## Component Responsibilities

| Component | Responsibility | Type | Modifies existing? |
|-----------|----------------|------|--------------------|
| `decision-logger.js` | Appends one structured record per hook event to `decision-log.jsonl` | New PostToolUse + Stop hook | No |
| `ml-analyzer.js` | Reads decision log, runs statistical model, writes `recommendations.json` | New offline SessionStart script | No |
| `config-writer.js` | Applies validated recommendations to config files atomically | New module (called by analyzer) | No |
| `decision-log.jsonl` | Per-project + global structured log of every hook decision + outcome | New data store | n/a |
| `adjustment-log.jsonl` | Append-only audit trail of every config change applied | New data store | n/a |
| `recommendations.json` | Latest analyzer output — overwritten each run, not a log | New data store | n/a |
| `defaults.json` | Cold-start priors with confidence scores | New data store | n/a |
| Existing 18 hooks | Unchanged — read config as before; behavior changes only when config changes | Existing | No |

---

## Recommended File Structure

```
~/.claude/hooks/
├── decision-logger.js          # NEW Phase 1: PostToolUse + Stop decision capture
├── ml-analyzer.js              # NEW Phase 2: offline analysis + recommendation writer
├── config-writer.js            # NEW Phase 3: atomic config update module
│
├── codex-router.js             # EXISTING — reads .claude/settings.json, unchanged
├── codex-review-gate.js        # EXISTING — reads config, unchanged
├── codex-plan-reviewer.js      # EXISTING — reads config, unchanged
├── codex-superpowers-plan-     # EXISTING — reads config, unchanged
│   reviewer.js
├── minimax-post-scan.js        # EXISTING — reads config, unchanged
├── minimax-compress.js         # EXISTING — reads config, unchanged
├── codex-token-logger.js       # EXISTING — token log source
├── codex-wave-validator.js     # EXISTING — wave boundary check
├── codex-wave-validator-       # EXISTING — background worker
│   worker.js
├── gsd-context-monitor.js      # EXISTING
├── gsd-workflow-guard.js       # EXISTING
├── gsd-prompt-guard.js         # EXISTING
├── gsd-check-update.js         # EXISTING
├── gsd-statusline.js           # EXISTING
├── claude-settings-guard.js    # EXISTING
├── codex-cost-reporter.js      # EXISTING
├── codex-global-aggregator.js  # EXISTING
├── codex-dashboard-generator.js# EXISTING
├── codex-handoff.js            # EXISTING
├── codex-multi-round-reviewer.js# EXISTING
└── ... (other existing)

~/.claude/dashboard/
├── global.jsonl                # EXISTING — token log aggregate
├── decision-log.jsonl          # NEW: global ML training signal (mirrors per-project)
├── adjustment-log.jsonl        # NEW: audit trail of every config change
├── recommendations.json        # NEW: latest analyzer output
└── defaults.json               # NEW: cold-start priors

~/<project>/.planning/
├── token-log.jsonl             # EXISTING — per-project token log
└── decision-log.jsonl          # NEW: per-project decision log
```

### Structure Rationale

- **Global `~/.claude/dashboard/decision-log.jsonl`**: Spans all projects. Enables cross-project
  patterns (e.g., `.sh` files trigger scan issues at 40% rate regardless of project).
- **Per-project `.planning/decision-log.jsonl`**: Enables per-project tuning. A project that is
  all markdown should have `routing_disabled: true` in its own `.claude/settings.json`, not globally.
- **`recommendations.json` is not a log**: It is overwritten each analyzer run. It represents the
  current best guess. The audit trail is `adjustment-log.jsonl`.
- **`defaults.json` is read-only at runtime**: Written at install time, never overwritten by the
  analyzer. The analyzer reads it for cold-start priors only.

---

## Data Flow: Full Self-Optimization Loop

```
Claude Code session opens
         │
         ▼
SessionStart hooks (sequential, registered in settings.json)
  1. gsd-check-update.js       (existing)
  2. codex-cost-reporter.js    (existing)
  3. codex-global-aggregator.js (existing)
  4. ml-analyzer.js             (NEW — reads decision-log from previous sessions)
         │
         ├── Reads ~/.claude/dashboard/decision-log.jsonl
         ├── Runs weighted statistics per tunable parameter
         ├── Writes ~/.claude/dashboard/recommendations.json
         └── If confidence >= 0.8: calls config-writer.js
                  │
                  ├── Reads current .claude/settings.json or .planning/config.json
                  ├── Merges recommendation
                  ├── Validates merged config (schema + safety bounds)
                  ├── Writes to .tmp.{pid} then fs.renameSync (atomic)
                  └── Appends record to adjustment-log.jsonl (audit)
         │
         ▼
User submits prompt → Claude calls Write / Edit / Bash
         │
         ▼
PreToolUse chain fires (existing, unchanged)
  codex-router.js reads .claude/settings.json
  → routing decision based on current config (which may have been updated at SessionStart)
         │
         ▼
Tool executes
         │
         ▼
PostToolUse chain fires (existing + new)
  gsd-context-monitor.js     → context warning if near limit
  codex-token-logger.js      → appends to token-log.jsonl
  codex-wave-validator.js    → wave boundary check
  minimax-post-scan.js       → bug scan, emits advisory text to tool_result
  minimax-compress.js        → compress large outputs, emits advisory text
  decision-logger.js  (NEW)  → reads tool_result advisory text from above hooks
         │                      infers routing/scan/compress decisions made
         │                      appends one record to .planning/decision-log.jsonl
         │                      appends one record to ~/.claude/dashboard/decision-log.jsonl
         │
         ▼
Stop hook fires (existing + new)
  codex-review-gate.js  → BLOCK or ALLOW, emits verdict text to tool_result
  decision-logger.js    → reads Stop tool_result, records review_verdict field
         │
         ▼ (if BLOCK: Claude gets another turn; decision-logger fires again next turn)
         │
         ▼
Session closes → next SessionStart processes accumulated decision-log.jsonl
```

---

## Decision Log Schema

Every record answers: "Under what conditions did this hook behave this way, and was the outcome good?"

```jsonc
{
  // Identity
  "timestamp":          "2026-04-03T10:00:00.000Z",
  "session_id":         "30092b41-eaa3-45bf-95b4-64463d5a2dbd",  // null if unknown
  "project":            "Claude_X_Codex",
  "cwd":                "/home/alucard/projects/Claude_X_Codex",

  // Tool context (what triggered the hooks)
  "tool_name":          "Write",             // Write | Edit | MultiEdit | Bash
  "file_ext":           ".js",              // extension of modified file, "" for Bash
  "file_path_hash":     "a3f9c2",           // first 6 chars of SHA-256(file_path)
  "diff_lines_changed": 42,                 // from git diff --stat; null if no git

  // codex-router.js signal
  "routing_advice_given": true,             // did router advise Codex delegation?
  "routing_disabled":     false,            // was routing disabled in config?

  // minimax-post-scan.js signal
  "scan_triggered":    true,               // did scan run (not skipped by threshold)?
  "scan_verdict":      "ISSUES_FOUND",     // "ISSUES_FOUND" | "CLEAN" | "SKIPPED" | "ERROR"
  "scan_severity":     "medium",           // "high" | "medium" | "low" | null
  "scan_issues_count": 2,                  // 0 if CLEAN

  // minimax-compress.js signal
  "compress_triggered": false,             // did compressor run?
  "compress_ratio":     null,              // original_chars / compressed_chars, or null

  // codex-review-gate.js signal (Stop hook — null if this is a PostToolUse record)
  "review_verdict":        null,           // "ALLOW" | "BLOCK" | null
  "review_block_category": null,           // "correctness" | "security" | "logic" | null

  // Cost signal (read from last token-log.jsonl record for this turn)
  "total_tokens_this_turn": 1240,
  "cost_usd_this_turn":     0.012,

  // Config snapshot at time of decision (for drift detection)
  "config_snapshot": {
    "routing_disabled":              false,
    "scan_skip_threshold":           5,
    "compress_tool_output_tokens":   2000,
    "compress_context_pct":          80
  }
}
```

**Field selection rationale:**

- `file_ext` + `diff_lines_changed`: the two strongest predictors of whether scan/review adds value.
  A 2-line `.md` change and a 200-line `.js` change should be treated differently.
- `config_snapshot`: without this, it is impossible to correlate a behavioral pattern to the config
  that was active when the pattern occurred. Essential for detecting if a prior adjustment worked.
- `file_path_hash`: avoids logging sensitive file names to a shared file while still enabling
  file-level duplicate detection.
- `review_block_category`: the most important outcome signal. A BLOCK on "security" when scan_verdict
  was "CLEAN" means the scan is missing something the review gate catches — actionable signal.
- All fields null-safe: the logger never blocks on missing data. Missing fields are simply null.

---

## ML Model: Weighted Running Statistics (Not Neural Network)

**Recommendation: use weighted running statistics with a confidence gate, not a neural network or
gradient-boosted model.**

Rationale:
- **Data volume is low**: a typical power user generates 50-200 hook events per day. Neural networks
  require thousands of labeled examples to generalize. Statistical rules on 50 examples are more
  reliable than a 10-parameter model trained on 50 examples.
- **Interpretability is critical**: when the system changes a config, the user must be able to
  understand why. "`.py` and `.sh` files have scan ISSUES_FOUND rate 40% vs `.md` at 2% →
  recommend lowering `scan_skip_threshold` for those extensions" is auditable. Neural network
  weights are not.
- **The decision space is small**: tunable config parameters are a handful of numeric thresholds
  and boolean flags — discrete optimization, not representation learning.
- **Cold start is handled by priors**: sensible defaults in `defaults.json` are easy to understand,
  easy to override, and require zero training data.

**What the statistical model computes (per tunable parameter P):**

```
signal_rate(P)  = count(events where P's hook fired)      / total_events
value_rate(P)   = count(events where P's hook found value) / count(events P's hook fired)
cost_rate(P)    = mean(cost_usd_this_turn where P's hook fired)

Recommendation logic:
  if value_rate < 0.05 AND cost_rate > HIGH_COST_THRESHOLD:
    → suggest disabling or raising threshold
    → reason: "fires often (N times), finds nothing (value_rate M%)"

  if value_rate > 0.30 AND signal_rate < 0.10:
    → suggest enabling or lowering threshold
    → reason: "rarely fires, but finds issues when it does (value_rate M%)"

confidence = min(event_count_for_P / 30, 1.0)
  → < 0.8: surface as advisory suggestion only, no auto-apply
  → >= 0.8: auto-apply if AUTO_APPLY_THRESHOLD set in analyzer config
```

---

## Cold Start Problem

**The problem:** On a fresh project, `decision-log.jsonl` is empty. The ML model has no signal.
Applying random defaults is dangerous — a bad routing config on day one poisons every session.

**Solution: curated defaults with explicit confidence scores (never auto-applied).**

`~/.claude/dashboard/defaults.json` (written at install time, never overwritten):

```json
{
  "_version": 1,
  "routing_disabled": {
    "value": false,
    "confidence": 0.5,
    "rationale": "Routing on by default. codex-router.js already exits silently if no project config."
  },
  "scan_skip_threshold": {
    "value": 5,
    "confidence": 0.5,
    "rationale": "Existing default from minimax-post-scan.js. No change until data accumulates."
  },
  "compress_tool_output_tokens": {
    "value": 2000,
    "confidence": 0.5,
    "rationale": "Existing default. Do not lower until project tool output size is known."
  },
  "compress_context_pct": {
    "value": 80,
    "confidence": 0.5,
    "rationale": "Existing default. Aggressive compression risks losing context."
  }
}
```

Cold-start behavior in `ml-analyzer.js`:

1. Count records in `decision-log.jsonl`.
2. If count < 30: read `defaults.json`, emit all values as recommendations with confidence 0.5.
   No auto-apply (0.5 < 0.8 threshold). Advisory context only: "ML optimizer: N/30 events collected.
   Using defaults. Tuning activates at 30 events."
3. 30-100 records: mixed mode — use defaults for parameters with fewer than 30 supporting events,
   use computed statistics for parameters with sufficient events.
4. >100 records: full statistics mode. Confidence gate still applies per-parameter.

This means the system starts conservatively and earns the right to adjust configs by accumulating
evidence. The user sees the count and knows when tuning will activate.

---

## Integration Points with All 18 Existing Hooks

### Hooks That Produce Signal for the ML Model

| Hook | Event | Signal captured | How captured |
|------|-------|-----------------|--------------|
| `codex-router.js` | PreToolUse | `routing_advice_given` | advisory text "delegate to Codex" in tool_result |
| `minimax-post-scan.js` | PostToolUse | `scan_triggered`, `scan_verdict`, `scan_severity`, `scan_issues_count` | advisory text markers in tool_result |
| `minimax-compress.js` | PostToolUse | `compress_triggered`, `compress_ratio` | "[Compressed from ~NK tokens]" header in tool_result |
| `codex-review-gate.js` | Stop | `review_verdict`, `review_block_category` | advisory text in Stop tool_result |
| `codex-token-logger.js` | PostToolUse | `total_tokens_this_turn`, `cost_usd_this_turn` | reads last record from token-log.jsonl |
| `gsd-context-monitor.js` | PostToolUse | `context_warning_fired` | advisory text in tool_result |
| `codex-wave-validator.js` | PostToolUse | `wave_validation_result` | advisory text in tool_result |
| `codex-plan-reviewer.js` | SubagentStop | `plan_review_block` | block decision in tool_result |
| `codex-superpowers-plan-reviewer.js` | SubagentStop | `superpowers_plan_block` | block decision in tool_result |
| `gsd-workflow-guard.js` | PreToolUse | `workflow_guard_fired` | advisory text in tool_result |
| `claude-settings-guard.js` | PreToolUse | `settings_guard_fired` | block decision in tool_result |
| `gsd-prompt-guard.js` | PreToolUse | `prompt_guard_fired` | advisory text in tool_result |

Hooks producing no direct signal (captured only as cost context via token-log.jsonl):
- `codex-cost-reporter.js` (SessionStart — produces reports, not per-turn decisions)
- `codex-global-aggregator.js` (SessionStart — aggregation, not decisions)
- `codex-dashboard-generator.js` (called via require, not a hook)
- `codex-handoff.js` (handoff utility)
- `gsd-check-update.js` (update check)
- `gsd-statusline.js` (status display)

### Hooks Whose Behavior the ML Model Can Tune

| Hook | Tunable Config Key | Config File | Effect of tuning |
|------|--------------------|-------------|------------------|
| `codex-router.js` | `codex.routing_disabled` | `.claude/settings.json` | Disable routing for a project |
| `minimax-post-scan.js` | `minimax.scan_skip_threshold` | `.claude/settings.json` | Lines-changed threshold below which scan is skipped |
| `minimax-compress.js` | `minimax.compress_tool_output_tokens` | `.claude/settings.json` | Token count threshold for compression |
| `minimax-compress.js` | `minimax.compress_context_pct` | `.claude/settings.json` | Context fill % threshold for compression |
| `gsd-workflow-guard.js` | `hooks.workflow_guard` | `.planning/config.json` | Enable/disable workflow guard |

### New Hook Registrations Required in settings.json

```json
"PostToolUse": [
  {
    "matcher": "Bash|Edit|Write|MultiEdit|Agent|Task",
    "hooks": [
      // ... existing 5 hooks unchanged (gsd-context-monitor, codex-token-logger,
      //      codex-wave-validator, minimax-post-scan, minimax-compress) ...
      {
        "type": "command",
        "command": "node \"/home/alucard/.claude/hooks/decision-logger.js\"",
        "timeout": 5
      }
    ]
  }
],
"Stop": [
  {
    "hooks": [
      {
        "type": "command",
        "command": "node \"/home/alucard/.claude/hooks/codex-review-gate.js\"",
        "timeout": 300
      },
      {
        "type": "command",
        "command": "node \"/home/alucard/.claude/hooks/decision-logger.js\"",
        "timeout": 5
      }
    ]
  }
],
"SessionStart": [
  {
    "hooks": [
      // ... existing gsd-check-update, codex-cost-reporter, codex-global-aggregator ...
      {
        "type": "command",
        "command": "node \"/home/alucard/.claude/hooks/ml-analyzer.js\"",
        "timeout": 10
      }
    ]
  }
]
```

---

## Separation of Concerns

```
Logging               Analysis              Adjustment
(decision-logger)     (ml-analyzer)         (config-writer)

Runs every turn       Runs once/session     Called by analyzer only
Advisory only         Advisory output       Write-only
Fail-silent always    Read-only input       Atomic + validated
No external calls     No hook I/O           No analysis
5s timeout            10s timeout           Sync, fast (< 100ms)
Append-only output    Overwrites recs.json  Append-only audit log
```

**Invariants:**
- `decision-logger.js` never modifies any config file.
- `ml-analyzer.js` never writes to hook inputs — only to `recommendations.json` and advisory stdout.
- `config-writer.js` never reads decision logs or computes statistics.
- None of the existing 18 hooks are modified or aware that an ML layer exists.
- If any new component crashes: existing 18 hooks continue working exactly as before.

---

## Atomic Config Update Pattern

Config files are read by multiple hooks on every invocation. Writing without atomic rename produces
torn reads.

```javascript
// config-writer.js — apply one recommendation atomically
function applyRecommendation(configPath, key, newValue, reason, confidence) {
  const current = JSON.parse(fs.readFileSync(configPath, 'utf8'));
  const previous = current[key];
  const updated = deepSet(current, key, newValue);  // handles nested keys like "codex.routing_disabled"

  // Validate before writing — schema check + safety bounds
  if (!isValidConfig(updated)) {
    throw new Error(`Config validation failed for key ${key}=${newValue}`);
  }

  // Atomic write: tmp file then rename (POSIX rename(2) is atomic on same filesystem)
  const tmp = configPath + '.tmp.' + process.pid;
  fs.writeFileSync(tmp, JSON.stringify(updated, null, 2) + '\n', 'utf8');
  fs.renameSync(tmp, configPath);  // atomic

  // Audit trail — append BEFORE applying so a crash mid-apply is detectable
  appendJsonl(ADJUSTMENT_LOG_PATH, {
    timestamp: new Date().toISOString(),
    config_file: configPath,
    key,
    previous_value: previous,
    new_value: newValue,
    reason,
    confidence,
    auto_applied: true,
  });
}
```

Safety bounds enforced in `isValidConfig()`:
- `scan_skip_threshold`: integer, 1–100 (never 0 — would scan every whitespace change)
- `compress_tool_output_tokens`: integer, 500–10000 (never below 500 — would compress tiny outputs)
- `compress_context_pct`: integer, 50–95 (never below 50 — compression at 50% context is already aggressive)
- `routing_disabled`: boolean only
- `workflow_guard`: boolean only

---

## Anti-Patterns

### Anti-Pattern 1: Modifying Existing Hooks to Emit Decision Signal

**What people do:** Add `appendJsonl(decisionLog, {...})` lines inside `codex-router.js`,
`minimax-post-scan.js`, etc.

**Why it's wrong:** Every hook has two responsibilities. When the decision schema changes, you
edit 7 hooks. When a hook fails, it is unclear if the failure was in the original job or in logging.
The fail-silent pattern becomes ambiguous.

**Do this instead:** Run `decision-logger.js` last in the PostToolUse chain. Parse the advisory
text that preceding hooks already emit. Each hook stays single-responsibility.

### Anti-Pattern 2: Auto-Applying Changes Without a Confidence Gate

**What people do:** Apply every ML recommendation immediately, regardless of sample size.

**Why it's wrong:** With 10 events, a 70% "scan finds nothing" rate may reflect a quiet week, not
a real pattern. On day 3, the system turns off your security scanner because you wrote markdown.

**Do this instead:** Hold all changes behind `confidence >= 0.8`. Below that, surface as advisory
context only. Let data accumulate to earn the right to auto-apply.

### Anti-Pattern 3: Putting the Analyzer in the PostToolUse Chain

**What people do:** Run the ML analyzer on every Write/Edit event to tune configs in real time.

**Why it's wrong:** PostToolUse hooks run inside the tool call latency budget. An analyzer
reading JSONL and computing statistics adds 100-500ms to every edit. Worse, config changes
in PostToolUse would not affect other hooks that already read config at chain start.

**Do this instead:** Analyze at SessionStart. Config changes apply at the next session open,
which is the correct granularity.

### Anti-Pattern 4: Writing Config Changes Without an Audit Trail

**What people do:** Overwrite `.claude/settings.json` with new values, no record of what changed.

**Why it's wrong:** When the ML model makes a bad recommendation that slips through validation,
there is no way to know what was changed or why. The user sees unexpected behavior with no
traceable cause.

**Do this instead:** Append one record to `adjustment-log.jsonl` before applying every change.
Record includes previous value, new value, reason, and confidence score. Rollback is two steps:
read the log, write the old value back with config-writer.

### Anti-Pattern 5: A Neural Network on Low-Volume Data

**What people do:** Install TensorFlow/PyTorch, train a model on session data, predict optimal
config values.

**Why it's wrong:** 50-200 events per day is statistical noise for a neural network. A model
with 10 parameters trained on 50 examples will overfit trivially and make worse predictions
than a simple threshold rule. The infrastructure overhead (Python runtime, dependencies, training
pipeline) is disproportionate.

**Do this instead:** Weighted running statistics with explicit confidence scores. Interpretable,
auditable, and calibrated to the data volume this system actually generates.

---

## Scaling Considerations

This is a single-user local CLI tool. The relevant scaling axis is data volume over time.

| Records in decision-log.jsonl | Analyzer behavior | SessionStart overhead |
|-------------------------------|-------------------|-----------------------|
| 0–29 | Cold start: defaults, no changes | < 1ms |
| 30–500 | Mixed mode, low-confidence suggestions | 5–20ms |
| 500–10,000 | Full statistics, auto-apply enabled | 20–100ms |
| 10,000–100,000 | Full statistics | 100ms–1s: add mtime-gated incremental read (same pattern as codex-global-aggregator.js) |
| 100,000+ | Same incremental read pattern | < 50ms with byte-offset seek |

The 10,000-record threshold takes approximately 3-6 months at typical usage. Implement the
mtime-gated incremental read in Phase 2 to avoid a retrofit.

---

## Build Order (dependency-first)

Dependencies drive the order. Each phase can be tested in isolation before the next begins.

```
Phase 1: Decision Capture
  No dependencies on later phases.
  → decision-logger.js (PostToolUse + Stop advisory hook)
  → decision-log.jsonl schema
  → defaults.json (cold-start priors)
  Validation: run a session, confirm records appear in
              .planning/decision-log.jsonl and
              ~/.claude/dashboard/decision-log.jsonl

Phase 2: Analysis Engine (depends on Phase 1 data)
  → ml-analyzer.js (statistics only, no config writes yet)
  → recommendations.json schema
  → mtime-gated incremental read (prevents 10k-record slowdown later)
  Validation: node ml-analyzer.js --dry-run
              confirm sensible recommendations from real Phase 1 data
              confirm advisory context output format

Phase 3: Config Writer + Audit Trail (depends on Phase 2 recommendations)
  → config-writer.js (atomic write-then-rename + safety validation)
  → adjustment-log.jsonl
  Validation: apply a known recommendation manually
              confirm atomic write (no torn file on concurrent read)
              confirm audit record appears in adjustment-log.jsonl
              confirm bad value rejected by isValidConfig()

Phase 4: SessionStart Integration (depends on all above)
  → register ml-analyzer.js in settings.json SessionStart chain (timeout: 10)
  → wire analyzer → config-writer with confidence gate (threshold: 0.8)
  Validation: open a session, confirm config changes apply for
              high-confidence recommendations;
              confirm no changes for low-confidence recommendations;
              confirm session opens in under 3 seconds total
```

**Why this order:**
- Phase 1 has no dependencies. It captures data regardless of whether analysis is built.
- Phase 2 requires real data from Phase 1 to be meaningful. Statistical models on synthetic data
  have hidden calibration bugs.
- Phase 3 is the highest-risk component (writes to files other hooks read). Test standalone before
  wiring to automatic triggers.
- Phase 4 wires everything together only after each component is verified standalone. A broken
  SessionStart hook delays every session open — it must be robust before registration.

---

## Sources

- Live source: `/home/alucard/.claude/hooks/` — all 18 hook scripts inspected; stdin/stdout
  patterns, advisory text formats, config key names, timeout budgets confirmed from source
- Live source: `/home/alucard/.claude/settings.json` — all hook registrations, event types,
  timeout values confirmed
- Live data: `/home/alucard/projects/Claude_X_Codex/.planning/token-log.jsonl` — JSONL schema
  confirmed; null session_id pattern confirmed (25% of records)
- Live source: `/home/alucard/.claude/hooks/codex-global-aggregator.js` — mtime-gated incremental
  read pattern reused for decision-log scaling
- Prior research: `.planning/research/SUMMARY.md` — write-then-rename and mtime guard already
  proven in this codebase
- Linux `rename(2)` POSIX specification — atomic same-filesystem rename on Linux confirmed
- WebSearch: Gemini CLI hooks (2026) — confirms hook-as-advisory-observer is established pattern
  across CLI AI tools; hook-based config feedback loops are an emerging standard

---

*Architecture research for: ML-driven self-optimization layer on Claude X Codex hook pipeline*
*Researched: 2026-04-03*
