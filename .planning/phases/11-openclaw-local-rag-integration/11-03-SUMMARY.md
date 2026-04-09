---
phase: 11-openclaw-local-rag-integration
plan: "03"
subsystem: seraphim-plugin
tags: [rag, context-injection, pipeline-orchestration, config, commands]
dependency_graph:
  requires: ["11-01", "11-02"]
  provides: ["RAG-05"]
  affects: ["run.md pipeline", "new-project onboarding", "operator tooling"]
tech_stack:
  added: []
  patterns: ["silent-skip on RAG unavailability (D-09)", "fire-and-forget config read before phase dispatch"]
key_files:
  created:
    - ~/.claude/plugins/seraphim/commands/reindex.md
  modified:
    - ~/.claude/plugins/seraphim/lib/config.js
    - ~/.claude/plugins/seraphim/commands/run.md
    - ~/.claude/plugins/seraphim/commands/new-project.md
decisions:
  - "RAG step named 6b-RAG inserted between 6a and 6b to preserve existing numbering"
  - "Step 5b used for new-project.md warm-up to avoid renumbering Step 6"
  - "reindex.md exits 0 on missing index so operator gets clear next-step guidance"
metrics:
  duration_minutes: 3
  completed_date: "2026-04-09"
  tasks_completed: 2
  files_changed: 4
---

# Phase 11 Plan 03: RAG Pipeline Integration Summary

RAG context injection wired into run.md Step 6b-RAG, rag config defaults added to config.js, /seraphim:reindex command created with --full rebuild support, and model warm-up step added to new-project.md for first-run latency avoidance.

## Tasks Completed

| Task | Name | Commit | Files |
|------|------|--------|-------|
| 1 | Update config.js with rag defaults and run.md with RAG injection | 4b66014 | lib/config.js, commands/run.md |
| 2 | Create /seraphim:reindex command and add warm-up step to new-project.md | 9741587 | commands/reindex.md, commands/new-project.md |

## What Was Built

**config.js** — `CONFIG_DEFAULTS` now includes a `rag` sub-object with four fields: `enabled` (true), `token_budget` (1500), `max_source_files` (200), `threshold` (0.75). The `validate()` function was extended to accept `rag` as a valid key with type-guard checking.

**run.md** — New Step 6b-RAG inserted between the loop-cap check (6a) and phase dispatch (6b). Before each phase executes, the step reads the RAG config, constructs a phase-appropriate query string, calls `query-knowledge.js`, and prepends a `## Prior Project Knowledge` block to the phase prompt if context is non-empty. Fully silent on any failure path.

**reindex.md** — New `/seraphim:reindex` slash command. Resolves project root the same way as run.md. Supports `--full` to delete and rebuild the SQLite index from scratch. Runs `rag-indexer.js --reindex` synchronously, reports index size via `du -h`, and reminds the operator that automatic re-indexing also occurs after each phase via the rag-post-phase hook.

**new-project.md** — Step 5b added immediately before the summary step. Downloads `Xenova/all-MiniLM-L6-v2` via `@huggingface/transformers` pipeline() and caches it to `~/.cache/huggingface/`. Exits 0 on failure with a user-friendly message so project init is never blocked.

## Decisions Made

- Inserted RAG step as `6b-RAG` rather than renumbering to avoid breaking any external references to step numbers in run.md.
- Warm-up step numbered `5b` in new-project.md to preserve the Step 6 summary step number.
- reindex.md's Step 4 reports missing index as a warning (not error) since an empty project produces no indexable files — not a failure condition.

## Deviations from Plan

None — plan executed exactly as written.

## Known Stubs

None. All four files are fully wired. RAG injection uses the real `query-knowledge.js` tool from Plan 02. reindex.md calls the real `rag-indexer.js` from Plan 01.

## Self-Check: PASSED

- FOUND: ~/.claude/plugins/seraphim/lib/config.js
- FOUND: ~/.claude/plugins/seraphim/commands/run.md
- FOUND: ~/.claude/plugins/seraphim/commands/reindex.md
- FOUND: ~/.claude/plugins/seraphim/commands/new-project.md
- FOUND: 11-03-SUMMARY.md
- FOUND: plugin commit 4b66014 (Task 1)
- FOUND: plugin commit 9741587 (Task 2)
