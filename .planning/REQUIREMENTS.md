# Requirements: Claude X Codex

**Defined:** 2026-04-02
**Core Value:** Every task goes to the model that's best at it — Opus for reasoning and architecture, Codex for fast execution — with cross-model review catching what either model misses alone.

## v1 Requirements

Requirements for initial release. Each maps to roadmap phases.

### Foundation

- [x] **FNDTN-01**: AGENTS.md spec file exists at repo root, giving Codex project context, conventions, and the hard rule that Codex never makes architectural decisions
- [x] **FNDTN-02**: Codex CLI can be invoked from Claude Code hooks via `codex exec --json` with timeout wrapper (300s max)
- [ ] **FNDTN-03**: PreToolUse hook intercepts Claude tool calls and routes Codex-appropriate tasks (clearly-defined implementation, test generation, bulk operations) to Codex CLI
- [x] **FNDTN-04**: Claude Code security is verified (version 2.0.65+ for CVE patches, API keys in env vars only)
- [x] **FNDTN-05**: Headless Codex CLI authentication works via API key (not ChatGPT web login) for hook-triggered invocations

### Review & Quality

- [ ] **REVW-01**: Stop hook review gate blocks Claude from finishing until Codex reviews the output, with `stop_hook_active` guard preventing infinite loops
- [ ] **REVW-02**: Cross-model code review works bidirectionally — Claude reviews Codex output AND Codex reviews Claude output
- [ ] **REVW-03**: Opus-Codex plan review loop (2-3 rounds) triggers before every GSD phase plan, with hard 3-round cap
- [ ] **REVW-04**: Opus-Codex plan review loop (2-3 rounds) triggers before every GSD individual task plan
- [ ] **REVW-05**: Opus-Codex plan review loop integrates into Superpowers planning/implementation design phases
- [ ] **REVW-06**: Review loop produces a typed handoff spec with decisions-not-taken section, and Opus has final authority after round 3

### Routing & Orchestration

- [x] **ROUT-01**: Opus remains sole model for architectural decisions and complex reasoning — enforced by hooks and AGENTS.md, not just convention
- [ ] **ROUT-02**: Global Claude hooks auto-route Codex-specialized tasks in general workflows (not just GSD/Superpowers)
- [ ] **ROUT-03**: Fallback routing gracefully degrades to Opus when Codex CLI rate-limits or fails
- [ ] **ROUT-04**: Codex CLI (subscription) preferred over API calls; API used only for quick model-to-model communication

### GSD Integration

- [ ] **GSD-01**: GSD plugin source modified to dispatch Codex validation at wave boundaries (post-wave-execution)
- [ ] **GSD-02**: Background Codex validation runs non-blocking during Claude execution, results available at natural stopping points
- [ ] **GSD-03**: GSD plan-phase workflow triggers the Opus-Codex review loop before plan finalization
- [ ] **GSD-04**: GSD execute-phase workflow routes clearly-defined implementation tasks to Codex where appropriate

### Superpowers Integration

- [ ] **SPWR-01**: Superpowers plugin source modified to use Codex during planning/implementation design phases
- [ ] **SPWR-02**: Superpowers plan review uses the same Opus-Codex review loop as GSD (2-3 rounds, 3-round cap)
- [ ] **SPWR-03**: Superpowers parallel agent dispatch can route hypothesis-testing threads to GPT-5.4-mini (via API) instead of spawning more Opus subagents

### Tracking & Reporting

- [x] **TRCK-01**: Every model call logged with: model name, task type, tokens_in, tokens_out, cost, timestamp
- [x] **TRCK-02**: Token tracking covers both Claude (from session JSONL) and Codex (from `--json` JSONL output)
- [ ] **TRCK-03**: Session cost report generated showing actual cost vs estimated Opus-only baseline cost
- [ ] **TRCK-04**: Cost reports written to `.planning/session-reports/YYYY-MM-DD.md` in human-readable format

## v2 Requirements

Deferred to future release. Tracked but not in current roadmap.

### Advanced Routing

- **ADVR-01**: Adaptive handoff spec generation — Opus auto-generates file-level specs for complex tasks, feature-level for simple ones
- **ADVR-02**: Rate-limit aware routing that detects Plus subscription limits and downshifts to API automatically
- **ADVR-03**: Cross-wave integration check routing full codebase diffs to Codex 1M context window

### Extended Integration

- **EXTI-01**: Superpowers parallel hypothesis testing via Codex-Spark (requires ChatGPT Pro upgrade to $200/mo)
- **EXTI-02**: Daily spend dashboard with 80% warning threshold

## Out of Scope

| Feature | Reason |
|---------|--------|
| Real-time streaming between models | Async handoff sufficient; streaming adds latency and complexity |
| Codex autonomous architecture decisions | Codex hallucinates APIs at higher rates; Opus must architect |
| Gemini/Llama/other model support | Scope limited to Anthropic + OpenAI; third model multiplies complexity |
| Web UI or dashboard | Terminal-first integration; CLI reports are sufficient |
| Universal auto-routing via prompt analysis | Keyword classification is fragile; explicit routing points are safer |
| Infinite review loops | Budget and reliability risk; capped at 3 rounds with human escalation |
| Modifying Claude Code core | Breaks on updates; use only hooks, skills, agents, plugins |
| Separate Codex session management CLI | `codex exec` and `codex resume` already cover this |

## Traceability

| Requirement | Phase | Status |
|-------------|-------|--------|
| FNDTN-01 | Phase 1 | Complete |
| FNDTN-02 | Phase 1 | Complete |
| FNDTN-03 | Phase 1 | Pending |
| FNDTN-04 | Phase 1 | Complete |
| FNDTN-05 | Phase 1 | Complete |
| ROUT-01 | Phase 1 | Complete |
| ROUT-03 | Phase 1 | Pending |
| ROUT-04 | Phase 1 | Pending |
| TRCK-01 | Phase 1 | Complete |
| TRCK-02 | Phase 1 | Complete |
| REVW-01 | Phase 2 | Pending |
| REVW-02 | Phase 2 | Pending |
| ROUT-02 | Phase 2 | Pending |
| GSD-01 | Phase 2 | Pending |
| GSD-02 | Phase 2 | Pending |
| GSD-03 | Phase 2 | Pending |
| GSD-04 | Phase 2 | Pending |
| REVW-03 | Phase 3 | Pending |
| REVW-04 | Phase 3 | Pending |
| REVW-05 | Phase 3 | Pending |
| REVW-06 | Phase 3 | Pending |
| SPWR-01 | Phase 3 | Pending |
| SPWR-02 | Phase 3 | Pending |
| SPWR-03 | Phase 3 | Pending |
| TRCK-03 | Phase 4 | Pending |
| TRCK-04 | Phase 4 | Pending |

**Coverage:**
- v1 requirements: 26 total
- Mapped to phases: 26
- Unmapped: 0

---
*Requirements defined: 2026-04-02*
*Last updated: 2026-04-02 after roadmap creation*
