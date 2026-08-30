# Execution-Plan Template

Use this after inspecting both the approved design and the current repository. Remove sections that do not affect execution.

```markdown
# {Feature name} — Execution Plan

- **Date:** YYYY-MM-DD
- **Status:** Ready | Blocked
- **Approved design:** `docs/plans/YYYY-MM-DD-{feature-name}.md`
- **Implementation scope:** {One-sentence outcome}

## Execution Contract

### Outcomes

- Observable result carried from the approved design.

### Invariants and Exclusions

- Behavior or boundary that must remain true.
- Work explicitly outside scope.

## Repository Findings

- `path/to/file.ext::Class.method` — current responsibility relevant to the work.
- Test, command, schema, or convention that constrains execution.

## Design Conflicts or Blockers

Write `None found` when inspection supports the design. Otherwise, for each conflict include:

- **Approved design says:** {decision or assumption}
- **Repository shows:** `{path::symbol}` and observed behavior
- **Impact:** blocked task or compatible choice
- **Decision needed:** only when execution would change architecture or scope

## Dependency Order

```text
Task 1 -> Task 2 -> Task 4
              \-> Task 3 -/
```

Use a list instead when the order is linear and obvious.

## Task 1 — {Checkable result}

- **Depends on:** None | Task N
- **Files:**
  - Modify: `exact/path.ext`
  - Create: `exact/path.ext`
  - Test: `exact/test_path.ext`
- **Symbols:** `Class.method`, `function_name`, `SchemaName`

### Change

State the precise behavior, contract, and integration point. Include concise code or pseudocode only for non-obvious logic:

```python
result = component.call(input)
```

### Verification

1. Add or update a test that demonstrates {behavior}.
2. Run `{repository-supported focused command}`.
   - Expected before implementation: {specific relevant failure, when using a test-first step}.
   - Expected after implementation: {pass signal or observable output}.
3. Run `{broader command}` when this task crosses an integration boundary.

### Done When

- [ ] Observable criterion, not an activity.
- [ ] Relevant automated checks pass.

## Task N — {Next checkable result}

Repeat the task shape. Split a task when it has multiple independently meaningful results, owners, deployment boundaries, or verification loops.

## Final Verification Matrix

| Design outcome or risk | Evidence | Command / environment | Expected signal |
|---|---|---|---|
| {Outcome} | {Test, eval, metric, or inspection} | `{command}` | {Result} |

## Non-Automated Checks

- Staging, human review, security review, AI evaluation, load test, migration rehearsal, or rollout observation that cannot run as an ordinary repository test.

## Completion Criteria

- [ ] Every task's done criteria are satisfied.
- [ ] Focused and repository-standard test suites pass.
- [ ] Approved design outcomes and invariants have evidence in the verification matrix.
- [ ] Required evaluation, migration, deployment, and rollback checks are complete.
- [ ] No unresolved design conflict was implemented around.
```

## Task Sizing

A useful task is small enough for one coding agent to implement and verify without making a new architectural decision. Split it if it:

- spans unrelated outcomes;
- crosses independent deployable units without an interface checkpoint;
- mixes schema migration, behavior, and rollout with no intermediate verification;
- says “update related files” instead of naming them;
- cannot define a focused success signal.

Do not fragment mechanical edits that share one result and one verification loop. “Bite-sized” means bounded and checkable, not artificially verbose.

## Precision Standard

Prefer:

```text
Modify `src/rag/pipeline.py::RagPipeline.answer` to pass `retrieval_query`
to both retrievers while retaining `request.message` for `PromptBuilder.build`.
```

Avoid:

```text
Update the RAG pipeline to support rewriting.
```

Exact line numbers are optional because they drift. Exact paths and symbols are required when the repository provides them.

