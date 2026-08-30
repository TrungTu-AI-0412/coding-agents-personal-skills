# Example: Query Rewriting for Hybrid RAG — Execution Plan

> Illustrative repository only. Its paths, symbols, and commands are examples of the precision expected after real inspection.

- **Date:** 2026-08-30
- **Status:** Ready
- **Approved design:** `docs/plans/2026-08-30-query-rewriting-hybrid-rag.md`
- **Implementation scope:** Add one validated rewrite before hybrid retrieval while preserving the original query for generation.

## Execution Contract

- `original_query` remains exact user input and is passed to `PromptBuilder.build`.
- One `retrieval_query` is shared by vector and keyword retrievers.
- Timeout, invalid output, or confidence below the configured threshold falls back to `original_query` with no request retry.
- The rewriter has no tools and receives only the bounded conversation window.
- No raw original or rewritten queries enter ordinary telemetry.

## Repository Findings

- `src/rag/pipeline.py::RagPipeline.answer` currently fans one query out to both retrievers and already joins them with `asyncio.gather`.
- `src/ai/model_gateway.py::ModelGateway.generate_structured` accepts a Pydantic response model, model name, and timeout.
- `src/bootstrap.py::build_rag_pipeline` explicitly constructs pipeline dependencies.
- Prompt resources use versioned YAML under `prompts/`; prompt snapshot tests live in `tests/prompts/`.
- Focused commands are `pytest tests/rag tests/prompts/test_query_rewrite_prompt.py -q` and `python -m evals.query_rewrite --dataset evals/query_rewrite/cases.jsonl`.

## Design Conflicts or Blockers

None found.

## Dependency Order

```text
Task 1 contract
   |
   +--> Task 2 model adapter + prompt --+
   |                                    |
   +--> Task 3 settings + composition --+--> Task 4 pipeline integration --> Task 5 evaluation/rollout evidence
```

## Task 1 — The rewrite contract rejects unusable output

- **Depends on:** None
- **Files:**
  - Create: `src/rag/query_rewriter.py`
  - Create: `tests/rag/test_query_rewriter.py`
- **Symbols:** `RewriteResult`, `QueryRewriter`, `select_retrieval_query`

### Change

Define the protocol and structured result from the approved design. Add a pure selector that returns the trimmed rewrite only when it is non-empty, at most 512 characters, and meets `min_confidence`; otherwise return the original query plus a typed fallback reason.

```python
def select_retrieval_query(
    original: str,
    result: RewriteResult,
    min_confidence: float,
) -> RetrievalQueryDecision: ...
```

### Verification

1. Cover accepted, empty, whitespace-only, too-long, exact-confidence, and below-confidence results.
2. Run `pytest tests/rag/test_query_rewriter.py -q`; expect import or assertion failure before implementation, then all cases passing.

### Done When

- [ ] Acceptance behavior is deterministic and provider-independent.
- [ ] Every rejection produces the unchanged original query and a typed reason.

## Task 2 — The model adapter returns a structured rewrite or a typed fallback

- **Depends on:** Task 1
- **Files:**
  - Create: `src/rag/model_query_rewriter.py`
  - Create: `prompts/query_rewrite.yaml`
  - Create: `tests/rag/test_model_query_rewriter.py`
  - Create: `tests/prompts/test_query_rewrite_prompt.py`
- **Symbols:** `ModelQueryRewriter.rewrite`, `QueryRewriteResponse`

### Change

Implement `QueryRewriter` using `ModelGateway.generate_structured`. Pass the original query and the already bounded conversation. Configure no tools and the approved timeout. Convert gateway timeout, provider error, and schema error into the existing typed AI-call error family so the pipeline can apply fallback.

The prompt must request only a standalone retrieval query and confidence. Keep the response schema in code and reference its version from prompt metadata.

### Verification

1. Mock the gateway and cover valid output, timeout, provider error, malformed output, and exact argument boundaries.
2. Snapshot the rendered prompt for an ambiguous follow-up and assert no tool definition is present.
3. Run `pytest tests/rag/test_model_query_rewriter.py tests/prompts/test_query_rewrite_prompt.py -q`.

### Done When

- [ ] The adapter uses one structured model call and no tools.
- [ ] Provider failures remain typed and contain no conversation content in their log fields.
- [ ] Prompt snapshot and adapter tests pass.

## Task 3 — Rewriting is configurable and wired through existing composition

- **Depends on:** Task 1
- **Files:**
  - Modify: `src/settings.py`
  - Modify: `src/bootstrap.py`
  - Modify: `tests/test_settings.py`
  - Modify: `tests/test_bootstrap.py`
- **Symbols:** `RagSettings`, `build_rag_pipeline`

### Change

Add the approved enable flag, model, timeout, and minimum confidence settings with rewriting disabled by default. Construct `ModelQueryRewriter` through `ModelGateway` and inject it into `RagPipeline`; use the repository's no-op implementation when disabled so pipeline branches remain testable.

### Verification

1. Cover defaults, validation bounds, enabled construction, and disabled no-op construction.
2. Run `pytest tests/test_settings.py tests/test_bootstrap.py -q`.

### Done When

- [ ] Existing deployments remain disabled without configuration changes.
- [ ] Enabled and disabled dependency graphs are covered by tests.

## Task 4 — Hybrid retrieval uses the rewrite while generation preserves user intent

- **Depends on:** Tasks 2 and 3
- **Files:**
  - Modify: `src/rag/pipeline.py`
  - Modify: `tests/rag/test_pipeline.py`
- **Symbols:** `RagPipeline.__init__`, `RagPipeline.answer`

### Change

Before starting retrieval, call the rewriter once. Apply `select_retrieval_query`; on typed rewriter error, choose `original_query` and record the fallback reason. Pass the selected `retrieval_query` to both retrievers. Continue passing the original request message and conversation to `PromptBuilder.build`.

Emit metadata-only telemetry: attempted, applied, fallback reason, confidence bucket, latency, model, and token counts supplied by the gateway.

### Verification

1. Add a successful integration test asserting one rewrite call, the same rewrite at both retrievers, and the exact original query at the prompt builder.
2. Add timeout, invalid, low-confidence, disabled, and empty-conversation cases asserting original-query fallback.
3. Assert telemetry contains no query or conversation text.
4. Run `pytest tests/rag/test_pipeline.py -q`, then `pytest tests/rag tests/prompts/test_query_rewrite_prompt.py -q`.

### Done When

- [ ] Both retrievers share one selected retrieval query.
- [ ] The answer prompt always retains exact user input.
- [ ] Every approved fallback keeps the request available without retry.
- [ ] Focused RAG and prompt suites pass.

## Task 5 — Evaluation and shadow-mode output support the rollout gate

- **Depends on:** Task 4
- **Files:**
  - Create: `evals/query_rewrite/cases.jsonl`
  - Create: `evals/query_rewrite/runner.py`
  - Modify: `src/rag/pipeline.py`
  - Modify: `docs/operations/rag-rollout.md`
- **Symbols:** `QueryRewriteEvaluator`, shadow result fields

### Change

Add the approved versioned evaluation cases and runner output for recall@k, nDCG, intent preservation, no-answer rate, latency, and model cost. In shadow mode, compute the rewrite and compare retrieval metadata without using rewritten results for generation. Document the existing flag change and rollback procedure; do not encode rollout percentages in application code.

### Verification

1. Run `python -m evals.query_rewrite --dataset evals/query_rewrite/cases.jsonl`; expect a machine-readable report with every approved metric and per-language slices.
2. Run `pytest tests/rag/test_pipeline.py -q`; expect shadow tests to show unchanged selected results plus comparison telemetry.

### Done When

- [ ] Evaluation output supports every design rollout gate.
- [ ] Shadow mode cannot change generated answers.
- [ ] Operations documentation names enable, monitor, and disable actions.

## Final Verification Matrix

| Outcome or risk | Evidence | Command / environment | Expected signal |
|---|---|---|---|
| Original query remains authoritative | Pipeline integration tests | `pytest tests/rag/test_pipeline.py -q` | Exact original reaches prompt builder |
| Rewrite improves retrieval without silent failure | Eval report and fallback tests | `python -m evals.query_rewrite --dataset evals/query_rewrite/cases.jsonl` | Metrics and fallback counts emitted |
| Bounded latency and cost | Gateway/pipeline tests plus shadow telemetry | Staging shadow cohort | Within approved budgets |
| No content telemetry | Event assertions | `pytest tests/rag -q` | Metadata-only payloads |

## Non-Automated Checks

- Review the prompt and evaluation cases for supported languages.
- Confirm provider residency before enabling each deployment region.
- Approve the evaluation report before moving from shadow to serving traffic.

## Completion Criteria

- [ ] All five tasks meet their done criteria.
- [ ] Focused and repository-standard suites pass.
- [ ] Offline and shadow evidence covers the approved quality, latency, cost, privacy, and intent-preservation gates.
- [ ] Rewriting remains disabled until rollout approval.

