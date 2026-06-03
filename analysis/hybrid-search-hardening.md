# Hybrid Search Hardening

Status: shipped on 2026-05-25 in `allknower`.

This note records the shipped RAG hardening work for AllKnower's hybrid retrieval path. It is a cross-service reference for operators and agents who need to understand the current search behavior without reading the original implementation plan.

## Current Search Path

AllKnower retrieval uses a hybrid pipeline:

1. LanceDB vector search returns semantic candidates.
2. LanceDB native FTS/BM25 returns keyword candidates.
3. Reciprocal Rank Fusion combines both result sets.
4. Optional OpenRouter rerank reorders the fused candidate pool.

The dedicated "Blackstone Keep" integration scenario confirms exact-name retrieval through the shipped hybrid path.

## Shipped Changes

- Added environment tuning knobs for vector candidate count, BM25 candidate count, RRF `k`, vector similarity threshold, rerank pool size, rerank document length, and rerank enablement.
- Extracted RRF scoring into `src/rag/rrf.ts` with unit coverage.
- Enriched retrieval logs with vector/BM25 candidate counts, RRF settings, and rerank settings.
- Added FTS index health tracking to `checkLanceDbHealth()`.
- Added `/brain-dump/stream` SSE keepalive comments after live integration exposed idle stream closure during long RAG, LLM, and write phases.
- Updated e2e env mocks so hybrid defaults are defined in test mode. Missing mock values previously produced `NaN` RRF and rerank values that serialized to invalid responses.

## Tunable Environment Fields

| Field | Purpose | Default behavior |
|---|---|---|
| `RAG_HYBRID_VECTOR_K` | Vector candidate count before fusion | `0` means `topK * 3` |
| `RAG_HYBRID_BM25_K` | BM25 candidate count before fusion | `0` means `topK * 3` |
| `RAG_HYBRID_RRF_K` | Reciprocal Rank Fusion smoothing constant | `60` |
| `RAG_VECTOR_SIMILARITY_THRESHOLD` | Minimum vector similarity score | `0.3` |
| `RAG_RERANK_TOP_N` | Candidate count sent to rerank and returned from rerank | `10` |
| `RAG_RERANK_DOC_MAX_CHARS` | Per-document character cap for rerank payloads | `2048` |
| `RAG_RERANK_ENABLED` | Enables or disables OpenRouter rerank | `"true"` |

## Deliberate Non-Decision

Postgres `tsvector` indexing was rejected for v1 because LanceDB native FTS is already shipped, tested, and integrated with vector search. Adding Postgres FTS would require a new model, migration, dual writes, backfill, and a parallel query path that duplicates the working LanceDB implementation.

If LanceDB FTS proves insufficient at larger scale, open a benchmark-backed plan before adding a second search index.

## Verification

The shipped change was validated with:

```bash
cd /Users/allmaker/projects/allcodex-aio/allknower
bun run check
```

Result: green. One existing `todo` remained for the real better-auth sign-in integration because deterministic seeded owner credentials were not available.
