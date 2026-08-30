# Example: Retrieval Score Threshold — Execution Plan

> Illustrative repository only. Its paths, symbols, and commands are examples of the precision expected after real inspection.

- **Date:** 2026-08-30
- **Status:** Ready
- **Approved design:** `docs/plans/2026-08-30-retrieval-score-threshold.md`
- **Implementation scope:** Filter normalized retrieval results before context construction and preserve the existing no-evidence path.

## Execution Contract

- Keep chunks with `score >= minimum_retrieval_score` and preserve their order.
- Use normalized relevance scores, not backend distance.
- Default threshold `0.0` preserves current behavior.
- Do not change retrieval, ranking, context, or fallback contracts.

## Repository Findings

- `src/settings.py::RagSettings` uses Pydantic settings and validates numeric ranges.
- `src/rag/answer_service.py::AnswerService.answer` is the only production caller that connects `Retriever.search` to `ContextBuilder.build`.
- `tests/rag/test_answer_service.py` uses `FakeRetriever` and spies on `ContextBuilder.build`.
- Focused tests run with `pytest tests/rag/test_answer_service.py tests/test_settings.py -q`.

## Design Conflicts or Blockers

None found.

## Task 1 — The threshold has a validated, backward-compatible setting

- **Depends on:** None
- **Files:**
  - Modify: `src/settings.py`
  - Modify: `tests/test_settings.py`
- **Symbols:** `RagSettings.minimum_retrieval_score`

### Change

Add a float setting with default `0.0` and inclusive bounds `0.0` and `1.0`, following existing Pydantic field conventions.

### Verification

1. Add tests for the default, both bounds, and values immediately outside both bounds.
2. Run `pytest tests/test_settings.py -q` before implementation; expect the new setting tests to fail because the field is absent.
3. Implement the field and rerun the command; expect all settings tests to pass.

### Done When

- [ ] Valid settings expose the configured value as a float.
- [ ] Out-of-range values fail settings construction.
- [ ] Existing configurations retain current retrieval behavior.

## Task 2 — Only qualified chunks reach context construction

- **Depends on:** Task 1
- **Files:**
  - Modify: `src/rag/answer_service.py`
  - Modify: `tests/rag/test_answer_service.py`
- **Symbols:** `AnswerService.__init__`, `AnswerService.answer`

### Change

Inject `RagSettings` using the service's existing constructor pattern. Between `Retriever.search` and `ContextBuilder.build`, retain chunks whose normalized score satisfies the inclusive threshold:

```python
qualified = [
    chunk
    for chunk in retrieved
    if chunk.score >= self._settings.minimum_retrieval_score
]
```

Pass `qualified` to the existing context builder. Do not add a separate fallback branch.

### Verification

1. Add focused cases for mixed scores, exact equality, all-below, empty retrieval, and preserved order.
2. In the all-below case, assert `ContextBuilder.build` receives `[]` and the existing no-evidence result is returned.
3. Run `pytest tests/rag/test_answer_service.py -q`; expect the new cases to fail before the filter and pass afterward.

### Done When

- [ ] The context builder never receives a below-threshold chunk.
- [ ] Equal scores are retained and ordering is stable.
- [ ] All-below and already-empty results use the same existing fallback behavior.

## Task 3 — Filtering emits metadata-only telemetry

- **Depends on:** Task 2
- **Files:**
  - Modify: `src/rag/answer_service.py`
  - Modify: `tests/rag/test_answer_service.py`
- **Symbols:** `AnswerService.answer`, `rag.retrieval_filtered` event

### Change

Emit the repository's structured event after filtering with `retrieved_count`, `qualified_count`, `threshold`, and optional `highest_score`. Do not include query text, chunk text, IDs, or metadata.

### Verification

1. Assert the mixed and empty-result events contain only the approved numeric fields.
2. Run `pytest tests/rag/test_answer_service.py -q`; expect the telemetry assertions and all behavior tests to pass.
3. Run the repository-standard unit suite: `pytest tests/rag tests/test_settings.py -q`.

### Done When

- [ ] Counts distinguish retrieval from qualification.
- [ ] Event payloads contain no content-bearing fields.
- [ ] The focused RAG and settings suites pass.

## Final Verification Matrix

| Outcome | Evidence | Command | Expected signal |
|---|---|---|---|
| Backward-compatible default | Settings and service tests | `pytest tests/test_settings.py tests/rag/test_answer_service.py -q` | All pass |
| Inclusive qualification and stable order | Parameterized service tests | Same command | Boundary cases pass |
| Existing no-evidence behavior | All-below and empty tests | Same command | Existing fallback returned |
| Metadata-only telemetry | Event payload assertions | Same command | No content fields present |

## Non-Automated Checks

- Run the approved offline retrieval evaluation before configuring a production threshold above `0.0`.

## Completion Criteria

- [ ] All three tasks meet their done criteria.
- [ ] The focused suites pass.
- [ ] The default deployment produces no behavior change.
- [ ] No threshold tuning is hard-coded into application logic.

