# Developer-Learning Guidance

The design should help a developer build a correct mental model without becoming a generic textbook.

## Teach Through the Change

Start from what the repository does today, then introduce the smallest conceptual delta:

```text
current invariant -> new pressure -> proposed boundary -> runtime consequence
```

For example, do not explain retrieval-augmented generation from first principles when the important lesson is why the original query remains the authority for generation while a rewritten query is used only for retrieval.

## Make Four Models Visible

Use the models that materially help the reader:

1. **Responsibility model:** which component decides, transforms, stores, or calls.
2. **Runtime model:** the ordered interactions for success, fallback, and recovery.
3. **Data model:** what data exists, how it changes, and which representation is authoritative.
4. **Operational model:** how the system is observed, evaluated, deployed, and restored.

A small change may need only one model. A distributed AI pipeline commonly needs all four.

## Explain Decisions at the Point of Consequence

Pair each non-obvious choice with its concrete effect:

```text
Choice: Rewrite only the retrieval query.
Consequence: Retrieval recall can improve without letting the rewriter alter user intent in the final prompt.
```

Avoid detached essays on patterns. If a concept does not alter a boundary, invariant, failure mode, or review decision, omit it.

## Use Concrete Scenarios

For E2 or E3, trace one representative input through the system. Prefer values that reveal the design:

- an ambiguous user query that the rewriter clarifies;
- a document containing text, tables, and images;
- retrieval results that all fall below the acceptance threshold;
- a worker retry after a model timeout.

Add a counterexample when it exposes an invariant or failure mode.

## Calibrate Code Detail

Use schemas, signatures, and pseudocode to make contracts precise. Include production code only when it is itself the clearest way to communicate a design-critical interface. Leave exact edit instructions and routine scaffolding to the execution plan.

## Mark Evidence and Uncertainty

- **Observed:** supported by a repository path, symbol, test, configuration, or authoritative document.
- **Inferred:** likely from surrounding code but not directly guaranteed.
- **Proposed:** part of this design and not current behavior.
- **Unresolved:** requires a reviewer decision or further discovery.

These labels are useful when evidence is mixed; they need not prefix every sentence.

## Keep Diagrams Useful

Use a diagram when it clarifies ownership, branching, sequence, or state. Label transformations and authoritative data. Do not use diagrams as decorative summaries of adjacent prose.

## Completion Test

A human reader should be able to answer:

- What problem are we solving, and what is outside scope?
- What happens to a representative request or data item?
- Which data and component are authoritative at each boundary?
- Why were the material choices made?
- What can fail, and what happens next?
- How will we know the AI behavior and the overall system are good enough?
- Where in the repository would manual implementation begin?

