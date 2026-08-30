# Example: Retrieval Score Threshold — Implementation Design

> This is an illustrative Level 1 plan. Paths and symbols belong to a hypothetical repository.

- **Status:** Draft
- **Date:** 2026-08-30
- **Engineering depth:** Level 1
- **Explanation depth:** E2

## Executive Summary

The RAG flow currently sends the top-ranked chunks to generation even when every match is weak. Add a configurable minimum relevance score between retrieval and context assembly. This prevents low-quality evidence from being presented as grounding while preserving the existing no-evidence response.

## Goals and Non-Goals

### Goals

- Exclude retrieved chunks whose normalized relevance score is below the configured threshold.
- Reuse the current no-evidence path when no chunks qualify.
- Make filtering observable without logging document content.

### Non-Goals

- Changing the embedding model, vector index, ranking algorithm, or top-k value.
- Selecting a universal threshold for every corpus.
- Replacing the existing no-evidence user experience.

## Current System

Observed repository flow:

- `src/rag/retriever.py::Retriever.search` returns `list[ScoredChunk]` in descending score order.
- `src/rag/answer_service.py::AnswerService.answer` passes that list directly to `ContextBuilder.build`.
- `ContextBuilder.build([])` already produces `NoEvidence`, which the service maps to the supported fallback answer.
- `src/settings.py::RagSettings` owns retrieval configuration.

```text
User query -> Retriever.search(top_k) -> ContextBuilder.build -> Generator
```

Ranking answers “which result is best among those found”; it does not answer “is any result good enough.”

## Proposed Design

Add a post-retrieval qualification step inside `AnswerService`, before context construction:

```text
Retriever.search
       |
       v
score >= minimum_retrieval_score?
       | yes                     | no
       v                         v
qualified chunks             discarded
       |
       v
ContextBuilder.build -> existing generation or no-evidence behavior
```

`ScoredChunk.score` is already the retriever's normalized, higher-is-better relevance value. The threshold uses that contract; it must not compare backend-specific raw distance.

### Configuration Contract

Add `minimum_retrieval_score: float` to `RagSettings` with a safe current-behavior default of `0.0` and validation for the inclusive range `[0.0, 1.0]`.

The comparison is inclusive:

```python
qualified = [chunk for chunk in chunks if chunk.score >= threshold]
```

A score exactly equal to the threshold remains eligible. This makes configuration boundaries predictable.

## Design Decisions

### Filter after retrieval, not inside the vector adapter

The threshold is product evidence policy, while the adapter is responsible for translating backend results into normalized scores. Keeping the filter in the application service preserves adapter reuse and makes the no-evidence transition explicit.

### Preserve the empty-result contract

The existing `ContextBuilder.build([])` behavior remains authoritative. Introducing a second fallback would create two user-visible no-evidence policies.

### Default to current behavior

A `0.0` default avoids silently reducing answer coverage at deployment. Corpus-specific threshold selection can happen through evaluation and configuration.

## Failure and Edge Behavior

- No retrieved chunks: unchanged no-evidence path.
- All scores below threshold: same no-evidence path.
- Mixed scores: preserve original rank order among qualified chunks.
- Invalid configuration: fail settings validation at startup.
- Missing scores: remains an adapter contract violation; this change does not invent a score.

## Observability and Evaluation

Record counts and score metadata only:

- `rag_retrieved_chunks_count`
- `rag_qualified_chunks_count`
- configured threshold
- highest normalized score when results exist

Do not log query text or chunk content. Offline evaluation should compare answer groundedness and no-answer rate before selecting a non-zero production threshold.

## Implementation Mapping

| Area | Repository location | Responsibility change |
|---|---|---|
| Settings | `src/settings.py::RagSettings` | Own and validate the threshold |
| Orchestration | `src/rag/answer_service.py::AnswerService.answer` | Filter before context assembly |
| Unit tests | `tests/rag/test_answer_service.py` | Cover boundary, mixed, and empty cases |
| Settings tests | `tests/test_settings.py` | Cover valid range and invalid startup values |

## Verification Strategy

- Unit tests demonstrate keep/drop behavior, inclusive equality, stable ordering, and existing no-evidence delegation.
- Settings tests demonstrate default compatibility and range validation.
- A small retrieval evaluation compares groundedness and no-answer rate using candidate thresholds; threshold tuning is rollout work, not code correctness.

## Risks and Mitigations

| Risk | Consequence | Mitigation |
|---|---|---|
| Score semantics differ by retriever | Incorrect filtering | Enforce normalized-score contract at adapters |
| Threshold is too high | Excessive no-answer responses | Default to `0.0`; tune using corpus evaluation |
| Content appears in telemetry | Privacy exposure | Emit counts and numeric scores only |

## Open Questions

- Which evaluation set and groundedness metric will authorize raising the production threshold above `0.0`?

