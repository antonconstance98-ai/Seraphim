# Milestones

## v1.0 Claude X Codex (Shipped: 2026-04-02)

**Phases completed:** 4 phases, 8 plans, 14 tasks

**Key accomplishments:**

- AGENTS.md Codex project brief and project-scope .claude/settings.json with routing opt-in OFF, security env vars pending user action
- Codex exec wrapper with 300s timeout + SIGTERM/SIGKILL and JSONL token logger with CODEX_RESULT marker detection appending to append-only token-log.jsonl
- One-liner:
- Stop hook review gate with ALLOW/BLOCK protocol, task-type-aware depth (D-12), and global opt-out routing replacing Phase 1 opt-in
- codex-wave-validator.js
- 2-round Opus-Codex plan review loop with constructive + adversarial Codex passes, early exit on clean plans, review-state.json persistence, and typed HANDOFF.md spec with Decisions Not Taken section
- One-liner:
- SessionStart hook that reads token-log.jsonl and generates a Markdown savings report comparing actual Codex costs against Opus-only baseline pricing

---
