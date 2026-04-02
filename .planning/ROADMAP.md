# Roadmap: Claude X Codex

## Overview

This project wires the OpenAI Codex CLI into the existing Claude Code + GSD + Superpowers setup so that Opus handles all reasoning and architecture while Codex handles fast, cheap execution and review. The build order is security-first: Phase 1 establishes the plumbing (AGENTS.md, hooks, auth, token log, routing rules) before any review logic is activated. Phase 2 adds the stop-hook review gate and all GSD integration points. Phase 3 adds the Opus-Codex plan review loop and Superpowers integration. Phase 4 closes the loop with cost reporting that proves the value of everything built.

## Phases

**Phase Numbering:**
- Integer phases (1, 2, 3): Planned milestone work
- Decimal phases (2.1, 2.2): Urgent insertions (marked with INSERTED)

Decimal phases appear between their surrounding integers in numeric order.

- [ ] **Phase 1: Foundation** - Security, plumbing, token tracking infrastructure, and routing rules — everything that must exist before any review logic is wired
- [ ] **Phase 2: Review Gate & GSD Integration** - Stop hook review gate, cross-model code review, and all GSD wave/plan integration points
- [ ] **Phase 3: Plan Review Loop & Superpowers** - Opus-Codex 2-3 round plan review loop (GSD + Superpowers) and Superpowers parallel agent routing
- [ ] **Phase 4: Cost Reporting** - Session cost reports proving savings vs Opus-only baseline

## Phase Details

### Phase 1: Foundation
**Goal**: The integration layer exists and is safe — Codex can be invoked from hooks, API keys are protected, every call is logged, and routing rules prevent cost runaway
**Depends on**: Nothing (first phase)
**Requirements**: FNDTN-01, FNDTN-02, FNDTN-03, FNDTN-04, FNDTN-05, ROUT-01, ROUT-03, ROUT-04, TRCK-01, TRCK-02
**Success Criteria** (what must be TRUE):
  1. AGENTS.md exists at repo root and Codex reads it for project context and conventions before executing any task
  2. A Claude Code hook can invoke `codex exec --json` with a 300s timeout wrapper and receive structured JSON output
  3. PreToolUse hook intercepts clearly-defined implementation and test-generation tasks and routes them to Codex instead of Opus
  4. Claude Code version is verified at 2.0.65+ with `CLAUDE_CODE_SUBPROCESS_ENV_SCRUB=1` set; no API keys appear in any hook script source
  5. Every Codex call appends a JSONL record to `.planning/token-log.jsonl` with model, task type, tokens in/out, cost, and timestamp — for both CLI and API calls
**Plans:** 3 plans

Plans:
- [x] 01-01-PLAN.md — Security baseline and Codex project brief (AGENTS.md, .claude/settings.json)
- [x] 01-02-PLAN.md — Codex execution wrapper and token tracking infrastructure
- [ ] 01-03-PLAN.md — Gap closure: PreToolUse routing hook and hook registration (FNDTN-03, ROUT-03, ROUT-04, TRCK-01, TRCK-02)

### Phase 2: Review Gate & GSD Integration
**Goal**: Every Claude session ends with a Codex review, and the GSD workflow has Codex checkpoints at plan-write and wave-boundary events
**Depends on**: Phase 1
**Requirements**: REVW-01, REVW-02, ROUT-02, GSD-01, GSD-02, GSD-03, GSD-04
**Success Criteria** (what must be TRUE):
  1. Stop hook blocks Claude from finishing until Codex reviews the output; `stop_hook_active` guard prevents infinite loops; ALLOW/BLOCK protocol is observed
  2. Cross-model review works both ways — Claude reviews Codex output and Codex reviews Claude output — with results injected as `additionalContext`
  3. GSD `plan-phase` workflow triggers a Codex review before the plan is finalized; Codex feedback appears in the plan file
  4. GSD wave-boundary Codex validation runs non-blocking during execution and results are available at natural stopping points
  5. All routing in general Claude workflows (not just GSD) goes through the global hook so Codex-specialized tasks are caught project-wide
**Plans**: TBD

### Phase 3: Plan Review Loop & Superpowers
**Goal**: Before any phase plan or task plan is finalized, a 2-3 round Opus-Codex review loop has run; Superpowers parallel agents can route to cheaper models
**Depends on**: Phase 2
**Requirements**: REVW-03, REVW-04, REVW-05, REVW-06, SPWR-01, SPWR-02, SPWR-03
**Success Criteria** (what must be TRUE):
  1. Opus-Codex review loop triggers before every GSD phase plan and every individual task plan, runs 2-3 rounds, and hard-caps at 3 with Opus retaining final authority
  2. Review loop produces a typed handoff spec that includes a `decisions_not_taken` section; round counter is persisted in `.planning/review-state.json`
  3. Superpowers `dispatching-parallel-agents` skill dispatches hypothesis-testing threads to `gpt-5.4-mini` via API instead of spawning additional Opus subagents
  4. Superpowers plan review uses the same 2-3 round, 3-cap Opus-Codex loop as GSD
**Plans**: TBD
**UI hint**: no

### Phase 4: Cost Reporting
**Goal**: After any session, the user can read a human-readable report showing actual spend vs what the same work would have cost using Opus alone
**Depends on**: Phase 3
**Requirements**: TRCK-03, TRCK-04
**Success Criteria** (what must be TRUE):
  1. A session cost report is generated reading `.planning/token-log.jsonl` and shows: actual total cost, estimated Opus-only equivalent cost, and savings amount/percentage
  2. Report is written to `.planning/session-reports/YYYY-MM-DD.md` in human-readable Markdown within seconds of session end
**Plans**: TBD

## Progress

**Execution Order:**
Phases execute in numeric order: 1 → 2 → 3 → 4

| Phase | Plans Complete | Status | Completed |
|-------|----------------|--------|-----------|
| 1. Foundation | 2/3 | In Progress |  |
| 2. Review Gate & GSD Integration | 0/TBD | Not started | - |
| 3. Plan Review Loop & Superpowers | 0/TBD | Not started | - |
| 4. Cost Reporting | 0/TBD | Not started | - |
