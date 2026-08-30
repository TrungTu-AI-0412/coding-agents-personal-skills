# Example: Query Rewriting for Hybrid RAG — Implementation Design

> This is an illustrative Level 2 plan. Paths and symbols belong to a hypothetical repository.

- **Status:** Draft
- **Date:** 2026-08-30
- **Engineering depth:** Level 2
- **Explanation depth:** E2

## Executive Summary

Short follow-up questions retrieve poorly because they depend on conversation context. Add a constrained query-rewriting component before hybrid retrieval. The rewritten query improves lexical and semantic recall, while the original user message remains authoritative for answer generation and audit logs.

## Goals and Non-Goals

### Goals

- Turn context-dependent follow-ups into standalone retrieval queries.
- Send the rewritten form to both vector and keyword retrieval.
- Fall back to the original query on rewriter timeout, invalid output, or low confidence.
- Measure retrieval and answer-quality changes independently.

### Non-Goals

- Letting the rewriter answer the question or add unsupported intent.
- Replacing the hybrid ranker.
- Persisting rewritten text as if it were user-authored.
- Adding multi-query expansion in the first version.

## Current System

Observed repository responsibilities:

- `src/chat/chat_service.py::ChatService.respond` loads recent conversation turns and invokes `RagPipeline.answer`.
- `src/rag/pipeline.py::RagPipeline.answer` sends `request.message` to both `VectorRetriever.search` and `KeywordRetriever.search`.
- `src/rag/hybrid_ranker.py::HybridRanker.merge` combines results and returns normalized ranked chunks.
- `src/rag/prompt_builder.py::PromptBuilder.build` uses the original request message plus grounded chunks.
- Provider calls use `src/ai/model_gateway.py::ModelGateway` with timeout and telemetry support.

```text
Current user message --------+------> vector retrieval ---+
                             +------> keyword retrieval --+--> rank --> context
                             +-------------------------------> answer prompt
```

For a follow-up such as “What about enterprise limits?”, the retrievers lack the earlier subject “Acme Search pricing,” even though generation later receives conversation history.

## Proposed Design

Introduce `QueryRewriter` as an application component called by `RagPipeline` before retrieval:

```text
Original message + bounded conversation context
                    |
                    v
               QueryRewriter
                 /       \
        valid rewrite   timeout / invalid / low confidence
              |                       |
              v                       v
       retrieval query          original message
              |
       +------+------+
       v             v
 vector search   keyword search
       +------ rank ------+
                        context

Original message + conversation + context -> answer prompt
```

The system carries two named values:

- `original_query`: exact user-authored message; authoritative for generation, display, and audit.
- `retrieval_query`: validated rewrite or the original query; used only to locate evidence.

This separation is the central invariant. Retrieval optimization must not silently redefine user intent.

## Request and Data Flow

For conversation history ending with “Compare Acme Search Pro and Enterprise,” followed by “What about limits?”:

1. `ChatService` loads the same bounded conversation window already used for generation.
2. `RagPipeline` requests a standalone retrieval query from `QueryRewriter`.
3. The rewriter returns `Acme Search Pro vs Enterprise usage and API limits` plus confidence `0.92`.
4. Deterministic validation accepts the result.
5. Vector and keyword retrievers receive the rewritten query.
6. The ranker merges results without knowing whether rewriting occurred.
7. `PromptBuilder` receives the original message, conversation, and retrieved chunks. It does not present the rewrite as user input.

If rewriting fails, steps 3–5 use `What about limits?`; answering remains available with current behavior.

## Interfaces and Contracts

```python
@dataclass(frozen=True)
class RewriteResult:
    query: str
    confidence: float

class QueryRewriter(Protocol):
    async def rewrite(
        self,
        original_query: str,
        conversation: Sequence[ConversationTurn],
    ) -> RewriteResult: ...
```

Acceptance rules are deterministic:

- trimmed query is non-empty;
- length is at most 512 characters;
- confidence is at least configured `query_rewrite_min_confidence`;
- output contains only the structured fields expected by the gateway parser.

The rewriter receives a bounded number of turns and no retrieved documents. It is not authorized to call tools.

## Design Decisions

### Rewrite once and share across retrievers

Both retrieval modes should search for the same interpreted intent. Independent rewrites would make ranking harder to debug and double model cost.

### Keep the ranker unaware of rewriting

The ranker combines normalized results. Adding rewrite policy there would mix query interpretation with ranking and expand its contract unnecessarily.

### Fail open to current retrieval

Rewriting improves quality but is not required for service availability. A short timeout and fallback bound added latency and preserve existing behavior during provider failure.

### Use structured model output plus deterministic validation

A schema prevents prose from becoming a search query; deterministic length and confidence rules keep model output inside an application-owned contract.

## Failure Modes and Recovery

| Failure | Behavior | Signal |
|---|---|---|
| Rewriter timeout or provider error | Use original query | fallback reason and latency |
| Invalid structured output | Use original query | validation reason |
| Low confidence | Use original query | confidence bucket |
| Empty conversation | Rewriter may return original query | rewrite-applied flag false |
| Retrieval failure | Existing retriever degradation policy | unchanged retriever telemetry |

No automatic retry occurs inside the request. A retry could exceed the latency budget and provides less value than using current behavior.

## Security, Privacy, and Safety

- The rewriter sees only the bounded conversation already authorized for the answer request.
- Rewritten queries are derived data: redact and retain them under the same policy as conversation data.
- Prompt instructions treat conversation text as data, not commands; structured output is validated before use.
- Tenant and document authorization remain enforced by both retrievers; rewriting cannot broaden the accessible corpus.

## Observability and Evaluation

Operational signals:

- rewrite attempted/applied/fallback counts;
- latency, model, token usage, and fallback reason;
- retriever result counts keyed by rewrite-applied status;
- no raw query text in ordinary telemetry.

Quality evaluation compares the baseline and new path on the same conversation dataset:

- retrieval recall@k and nDCG;
- grounded answer correctness;
- intent-preservation failures;
- no-answer rate;
- p50/p95 added latency and per-request model cost.

Rollout requires no statistically meaningful regression in intent preservation and agreed improvements in follow-up retrieval recall.

## Implementation Mapping

| Area | Repository location | Responsibility change |
|---|---|---|
| Contract | `src/rag/query_rewriter.py` | Define rewrite interface and validated result |
| Provider implementation | `src/rag/model_query_rewriter.py` | Prompt, structured model call, and parsing |
| Orchestration | `src/rag/pipeline.py::RagPipeline.answer` | Select retrieval query and preserve original |
| Composition | `src/bootstrap.py` | Construct and inject the rewriter |
| Configuration | `src/settings.py::RagSettings` | Model, timeout, confidence, and rollout flag |
| Prompt | `prompts/query_rewrite.yaml` | Versioned rewrite instruction and output schema |
| Tests | `tests/rag/` and `evals/query_rewrite/` | Contracts, fallbacks, integration, and quality set |

## Rollout

1. Deploy with `query_rewrite_enabled=false`.
2. Run offline evaluation against the versioned prompt and model configuration.
3. Enable shadow mode to record decisions and retrieval metrics without changing results.
4. Enable for a small tenant cohort, monitoring latency, fallbacks, intent preservation, and answer quality.
5. Expand or disable through configuration. No stored-data migration is required.

## Risks and Mitigations

| Risk | Consequence | Mitigation |
|---|---|---|
| Rewrite changes intent | Irrelevant or misleading evidence | Preserve original for generation; confidence gate; intent eval |
| Added model call increases latency/cost | Slower, more expensive answers | Small model, short timeout, single rewrite, rollout budget |
| Conversation leaks into telemetry | Privacy exposure | Derived-data policy and metadata-only metrics |
| Quality differs by language | Uneven retrieval gains | Segment evaluation and rollout metrics by language |

## Open Questions

- What confidence threshold meets the intent-preservation target on the approved evaluation set?
- Does the provider support the required data residency for every deployment region?

