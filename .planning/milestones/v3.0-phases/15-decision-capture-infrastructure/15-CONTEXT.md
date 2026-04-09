# Phase 15: Decision Capture Infrastructure - Context

**Gathered:** 2026-04-03
**Status:** Ready for planning

<domain>
## Phase Boundary

This phase adds structured logging of every routing and review decision across the hook pipeline. It enriches the existing JSONL schema with outcome tracking and latency, expands the task-type taxonomy from 4 to 12 categories, creates a dismiss command for false-positive feedback, and adds a freeze flag to disable all adaptive behavior. Pure data capture — no ML, no analysis, no config modification.

</domain>

<decisions>
## Implementation Decisions

### Decision Log Schema
- **D-01:** Claude's discretion on whether to create a new `decision-log.jsonl` or extend `token-log.jsonl`. Choose based on separation of concerns and query patterns — billing data vs training signals have different write frequencies and consumers.
- **D-02:** Signal capture from upstream hooks must use the best long-term approach, not the easiest. Research flagged advisory text parsing as brittle (breaks silently if format changes). A shared state mechanism (e.g., per-invocation temp file or structured output contract between hooks) is preferred over parsing advisory text prefixes, even though it requires modifying existing hooks. Durability over expedience.
- **D-03:** Log both `model_latency_ms` (API/CLI call time only) and `hook_latency_ms` (total hook execution time) as separate fields. Both signals are valuable — model latency for routing optimization, hook latency for performance debugging.

### Dismiss Workflow
- **D-04:** `/gsd:dismiss-last` dismisses the most recent BLOCK event only (not scans). Clear scope, easy to reason about.
- **D-05:** After dismiss, log the event AND show immediate feedback: "Dismissed 2/3 times for this rule — one more and it will be suppressed." User needs visible progress into the learning process.

### Task-Type Taxonomy
- **D-06:** Expand from 4 to 12 categories. Claude's discretion on exact categories and detection method — hook event type, tool name, file context (extensions/paths) are all available signals. Research suggested: refactor, explain, security-scan, plan-review, doc-update, test-write, architecture, debug, implementation, review, bulk-ops, hook-dev.

### Freeze Mode
- **D-07:** `adaptive: false` disables everything adaptive — auto-tuning, noise profiles, routing weight adjustments. Clean escape hatch back to static v2.0 rules.
- **D-08:** Toggle via `/gsd:freeze` and `/gsd:unfreeze` commands only. User doesn't need to know where the flag lives — it's an implementation detail.

### Claude's Discretion
- Log schema location (D-01): new file vs extending existing, based on codebase patterns
- Signal capture mechanism (D-02): shared state approach preferred, specific implementation TBD
- Task-type taxonomy categories and detection rules (D-06): hook event + tool name + file context

### General Principle
- **D-09:** When choosing between approaches, pick the best long-term option, not the easiest to implement. This applies to all decisions in this phase and downstream phases.

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Existing Hook Infrastructure
- `~/.claude/hooks/codex-token-logger.js` — Current JSONL schema, task_type field (4 values), additionalContext format
- `~/.claude/hooks/codex-review-gate.js` — BLOCK/ALLOW verdict format, block_summary field, dual model review
- `~/.claude/hooks/minimax-post-scan.js` — BUG SCAN advisory format, PostToolUse integration
- `~/.claude/hooks/codex-router.js` — Routing advisory format, PreToolUse integration
- `~/.claude/hooks/codex-pricing.js` — Centralized pricing for cost computation
- `~/.claude/hooks/codex-global-aggregator.js` — Global JSONL aggregation pattern (mtime-gated, dedup)

### Research
- `.planning/research/STACK.md` — better-sqlite3, simple-statistics, ml-regression recommendations
- `.planning/research/ARCHITECTURE.md` — Three-component architecture (logger, analyzer, config-writer)
- `.planning/research/PITFALLS.md` — False positive suppression trap, advisory text format contracts
- `.planning/research/SUMMARY.md` — Synthesized research with phase ordering rationale

### Configuration
- `~/.claude/settings.json` — Hook registrations, event types, timeouts
- `.planning/config.json` — GSD workflow configuration

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `codex-token-logger.js`: Established JSONL append pattern with `fs.appendFileSync` — reuse for decision log
- `codex-global-aggregator.js`: Mtime-gated incremental reads, dedup via Set — pattern for global decision log aggregation (Phase 18)
- `codex-pricing.js`: `computeCodexCostStrict()` and model pricing constants — reuse for cost fields in decision records
- Advisory text prefixes: `[Codex Token Log]`, `[BUG SCAN]`, `Codex plan review`, `Codex reviewed: PASS` — current parsing targets (but D-02 prefers shared state over parsing)

### Established Patterns
- All hooks use synchronous execution with JSON stdin/stdout
- Advisory output via `additionalContext` in `hookSpecificOutput`
- Append-only JSONL files per-project at `.planning/token-log.jsonl`
- Hook chain ordering matters — token-logger runs after exec hooks, scanner runs after write hooks

### Integration Points
- `decision-logger.js` will be a new PostToolUse + Stop hook running last in each chain
- Must integrate with `codex-token-logger.js` (reads its output or shares state)
- Must integrate with `codex-review-gate.js` (captures BLOCK/ALLOW verdicts)
- Must integrate with `minimax-post-scan.js` (captures scan results)
- `/gsd:dismiss-last` will be a new GSD command (skill) reading the decision log
- Freeze flag checked by all adaptive hooks (future phases) at the start of execution

</code_context>

<specifics>
## Specific Ideas

- Dismiss feedback should show progress: "Dismissed 2/3 times for this rule — one more and it will be suppressed"
- The Codex false-positive blocking on non-code conversation responses (observed during v3.0 milestone setup) is the canonical example of what the dismiss+noise profile system should eventually suppress
- User wants the best long-term approach, not shortcuts — shared state between hooks is preferred over parsing advisory text even if it means touching existing hook files

</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope

</deferred>

---

*Phase: 15-decision-capture-infrastructure*
*Context gathered: 2026-04-03*
