# Phase 11: OpenClaw Local RAG Integration - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md — this log preserves the alternatives considered.

**Date:** 2026-04-08
**Phase:** 11-openclaw-local-rag-integration
**Areas discussed:** RAG architecture approach, What gets indexed, OpenClaw research scope, Query integration

---

## RAG Architecture Approach

| Option | Description | Selected |
|--------|-------------|----------|
| Local embeddings + vector search | sentence-transformers on GPU, local vector DB. Best semantic matching. | |
| Keyword/BM25 search | Full-text search, no embeddings. Works on CPU, simpler. | |
| Hybrid (recommended) | BM25 default, upgrade to embeddings when GPU available. | |
| Research first | Don't commit — research OpenClaw first, then decide. | |

**User's choice:** Whatever the most accurate and efficient approach is for best quality. Do not worry about GPU constraints.
**Notes:** User explicitly said GPU constraints are not a concern — go for best quality.

---

## What Gets Indexed

| Option | Description | Selected |
|--------|-------------|----------|
| Pipeline outputs | Phase output files (vision.md, judgment.md, etc.) | ✓ |
| Decision history | decisions.jsonl, CONTEXT.md, discussion logs | ✓ |
| Source code | Actual project code files | ✓ |
| Everything in .seraphim/ | Complete project knowledge base | |

**User's choice:** Pipeline outputs, decision history, and source code. But honestly not super sure — research and see what the best hygiene is.
**Notes:** User is open to research findings on optimal indexing strategy.

---

## OpenClaw Research Scope

| Option | Description | Selected |
|--------|-------------|----------|
| Deep reverse-engineer | Clone repo, read RAG implementation in detail | |
| Architecture overview | Study docs and high-level architecture. Adapt pattern conceptually. | ✓ |
| Broad survey | Compare OpenClaw with Cursor, Continue.dev, Aider | |

**User's choice:** Architecture overview level.

---

## Query Integration

| Option | Description | Selected |
|--------|-------------|----------|
| Automatic context injection | System queries RAG and injects into phase prompt. Phases don't know RAG exists. | |
| Explicit tool call | Phases call query_knowledge tool when they need context. | |
| Both (recommended) | Automatic injection of high-relevance context + explicit tool for deep lookups. | ✓ |

**User's choice:** Both — automatic injection plus explicit tool calls.

---

## Claude's Discretion

- Embedding model choice
- Vector database selection
- Chunking strategy
- Relevance threshold for automatic injection
- Index update triggers
- Token budget for RAG context per phase

## Deferred Ideas

None — discussion stayed within phase scope
