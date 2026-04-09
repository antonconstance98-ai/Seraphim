# Phase 15: Decision Capture Infrastructure - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md — this log preserves the alternatives considered.

**Date:** 2026-04-03
**Phase:** 15-decision-capture-infrastructure
**Areas discussed:** Decision log schema, Dismiss workflow, Task-type taxonomy, Freeze mode UX

---

## Decision Log Schema

| Option | Description | Selected |
|--------|-------------|----------|
| New decision-log.jsonl (Recommended) | Separate file for decision signals — keeps token-log.jsonl clean, different write frequency | |
| Extend token-log.jsonl | Add outcome/dismissed/committed fields to existing records | |
| You decide | Claude picks the best approach based on codebase patterns | ✓ |

**User's choice:** You decide
**Notes:** None — deferred to Claude's judgment

---

| Option | Description | Selected |
|--------|-------------|----------|
| Parse advisory text (Recommended) | decision-logger.js runs last, parses advisory text from preceding hooks. Zero changes to existing hooks but brittle. | |
| Shared state file | Hooks write to a per-invocation temp file, decision-logger reads it. More robust but requires modifying existing hooks. | |
| You decide | Claude picks the best approach | |

**User's choice:** "Whatever is the best option not the easiest, best long term approach"
**Notes:** Explicit principle — durability over expedience. This overrides the "recommended" advisory-text-parsing option. Shared state is likely the better long-term choice.

---

| Option | Description | Selected |
|--------|-------------|----------|
| Model call only (Recommended) | Measures just the API/CLI call time | |
| End-to-end hook time | Measures total hook execution including file I/O, parsing, logging | |
| Both | Log model_latency_ms and hook_latency_ms separately | ✓ |

**User's choice:** Both
**Notes:** More data for ML layer — model latency for routing, hook latency for debugging.

---

## Dismiss Workflow

| Option | Description | Selected |
|--------|-------------|----------|
| Last block event only | Only dismisses the most recent BLOCK from review gate or plan reviewer | ✓ |
| Let user pick | Show recent blocks and scans, let user select which to dismiss | |
| Last block + last scan | Dismiss both the most recent BLOCK and the most recent BUG SCAN advisory | |

**User's choice:** Last block event only
**Notes:** Clear scope, easy to reason about.

---

| Option | Description | Selected |
|--------|-------------|----------|
| Log only (Recommended) | Just record the dismiss — noise profile processing in Phase 16 | |
| Log + immediate feedback | Log AND show count like "Dismissed 2/3 times for this rule" | ✓ |

**User's choice:** Log + immediate feedback
**Notes:** User wants visible progress into the learning process. Aligns with known preference for seeing what the system is doing.

---

## Task-Type Taxonomy

| Option | Description | Selected |
|--------|-------------|----------|
| Hook event + tool name | Simple rules from hook payload data. ~90% accuracy. | |
| Hook event + file context | Also checks file extensions and paths. ~95% accuracy but more rules. | |
| You decide | Claude picks the best detection approach | ✓ |

**User's choice:** You decide
**Notes:** Claude's discretion on taxonomy detection approach.

---

## Freeze Mode UX

| Option | Description | Selected |
|--------|-------------|----------|
| Everything adaptive | Disables auto-tuning, noise profiles, and all learned behavior | ✓ |
| Auto-tuning only | Disables config writes but keeps noise profiles active | |
| Granular toggles | Separate flags for each adaptive feature | |

**User's choice:** Everything adaptive
**Notes:** Clean escape hatch back to static v2.0 rules.

---

| Option | Description | Selected |
|--------|-------------|----------|
| Command + manual | /gsd:freeze and /gsd:unfreeze commands + direct settings.json editing | |
| Command only | Only via /gsd:freeze command — flag is implementation detail | ✓ |
| You decide | Claude picks the best approach | |

**User's choice:** Command only
**Notes:** User doesn't need to know where the flag lives.

---

## Claude's Discretion

- Decision log schema location (new file vs extending token-log.jsonl)
- Signal capture mechanism (shared state preferred per user principle)
- Task-type taxonomy categories and detection rules

## Deferred Ideas

None — discussion stayed within phase scope
