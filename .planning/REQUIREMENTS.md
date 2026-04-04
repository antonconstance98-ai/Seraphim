# Requirements — v3.0 Adaptive Intelligence

## Decision Capture (DCAP)

- [ ] **DCAP-01**: Every model call logs outcome (accepted/dismissed/committed) alongside existing token data
- [ ] **DCAP-02**: Every model call logs latency_ms for performance trend analysis
- [ ] **DCAP-03**: Task-type taxonomy expanded from 4 to 12 categories for routing accuracy
- [ ] **DCAP-04**: User can dismiss a false-positive review flag via /gsd:dismiss-last, creating explicit negative training signal
- [ ] **DCAP-05**: User can freeze adaptive behavior via adaptive:false flag, reverting to static rules instantly

## Analysis Engine (ANLZ)

- [ ] **ANLZ-01**: System builds per-project noise profiles, suppressing review rules after 3 consecutive dismissals in 30 days
- [ ] **ANLZ-02**: System detects implicit positive signal when Claude's output is committed to git without edits
- [ ] **ANLZ-03**: System supports per-project routing weights keyed by project path prefix
- [ ] **ANLZ-04**: Statistical analyzer runs at SessionStart computing weighted metrics across all decision data

## Auto-Tuning (TUNE)

- [ ] **TUNE-01**: Config writer applies threshold changes atomically (write-then-rename) with hard parameter clamps
- [ ] **TUNE-02**: Every auto-adjustment logged to adjustment-log.jsonl with before/after values and confidence score
- [ ] **TUNE-03**: Auto-apply gated behind confidence >= 0.8 AND 30+ events per parameter — read-only below threshold
- [ ] **TUNE-04**: Routing decision audit log shows why each call was routed to a specific model

## Cross-Project Intelligence (XPRJ)

- [ ] **XPRJ-01**: Global aggregator collects decision logs from all projects with three-model router installed
- [ ] **XPRJ-02**: System generates hypotheses from cross-project data patterns (e.g. "MiniMax outperforms Codex on security reviews")
- [ ] **XPRJ-03**: System designs experiments to test hypotheses and presents them for user approval before running
- [ ] **XPRJ-04**: System can propose implementation tweaks to specific projects to gather targeted data

## Observability (OBSV)

- [ ] **OBSV-01**: Dashboard panel shows dismiss rate, false-positive trend, and routing efficiency over time
- [ ] **OBSV-02**: Dashboard shows active hypotheses, experiment status, and cross-project insights

## Future Requirements (v4+)

- Review quality confidence scores (breaking schema change — needs 500+ labeled records)
- ML-trained task classifier (only if keyword/regex accuracy < 85%)
- Budget guardrail auto-downgrade (deferred from v3.0)

## Out of Scope

- Neural networks / deep learning — data volume too small; statistical methods only
- Embeddings-based semantic router — keyword classifier sufficient for 12 categories
- Real-time mid-session adaptation — hooks are stateless; session-to-session only
- Explicit per-call user ratings — annotation fatigue; implicit signals only
- A/B testing framework — single-user; sequential comparison sufficient

## Traceability

| REQ-ID | Phase | Plan | Status |
|--------|-------|------|--------|
| DCAP-01 | Phase 15 | — | Pending |
| DCAP-02 | Phase 15 | — | Pending |
| DCAP-03 | Phase 15 | — | Pending |
| DCAP-04 | Phase 15 | — | Pending |
| DCAP-05 | Phase 15 | — | Pending |
| ANLZ-01 | Phase 16 | — | Pending |
| ANLZ-02 | Phase 16 | — | Pending |
| ANLZ-03 | Phase 16 | — | Pending |
| ANLZ-04 | Phase 16 | — | Pending |
| TUNE-01 | Phase 17 | — | Pending |
| TUNE-02 | Phase 17 | — | Pending |
| TUNE-03 | Phase 17 | — | Pending |
| TUNE-04 | Phase 17 | — | Pending |
| XPRJ-01 | Phase 18 | — | Pending |
| XPRJ-02 | Phase 18 | — | Pending |
| XPRJ-03 | Phase 18 | — | Pending |
| XPRJ-04 | Phase 18 | — | Pending |
| OBSV-01 | Phase 18 | — | Pending |
| OBSV-02 | Phase 18 | — | Pending |
