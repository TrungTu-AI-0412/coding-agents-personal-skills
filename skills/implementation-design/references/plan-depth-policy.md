# Plan-Depth Policy

Select depth on two independent axes. Engineering depth describes the scope of the change; explanation depth describes how much teaching the intended reader needs. State both near the top of the design.

## Engineering Depth

### Level 1 — Local change

Use when behavior changes inside one component and existing architecture remains intact.

Typical signals:

- one local control-flow or configuration change;
- no new persistent data or cross-service contract;
- failure behavior already exists;
- a small number of files and tests are affected.

Cover the behavior delta, insertion point, configuration or interface impact, edge cases, and focused tests. A compact flow is usually enough.

AI example: filter retrieved chunks below a configurable relevance threshold before context assembly.

### Level 2 — Feature integration

Use when a feature crosses components or introduces a meaningful new runtime path without changing the system's fundamental ownership model.

Typical signals:

- several layers or components coordinate;
- a new abstraction, provider, prompt, tool, cache, or evaluation path is added;
- request or data contracts change;
- observability, rollout, latency, or cost needs explicit treatment.

Cover component responsibilities, request and data flows, contracts, failure modes, configuration, rollout, evaluation, and implementation mapping.

AI example: add query rewriting to a hybrid RAG flow while keeping the original user query authoritative for answer generation.

### Level 3 — Architectural change

Use when the change alters system boundaries, ownership, durability, consistency, orchestration, deployment, or operational responsibility.

Typical signals:

- new service, worker, event, durable workflow, or storage boundary;
- asynchronous or distributed state transitions;
- migration or compatibility concerns;
- security, tenancy, replay, idempotency, or recovery becomes central;
- several teams or deployable units may be affected.

Cover system context, ownership, state transitions, contracts, sequencing, consistency, failure recovery, security, observability, capacity, deployment, migration, evaluation, alternatives, and decision records.

AI example: move multimodal document ingestion from a synchronous API request into an asynchronous, resumable pipeline with model-specific workers and durable job state.

## Explanation Depth

### E1 — Concise

For readers already familiar with the codebase and concepts. Explain repository-specific behavior and non-obvious decisions. Avoid introductory material.

### E2 — Guided

Default for mixed-experience teams. Define important terms once, pair flows with brief interpretation, and explain why major boundaries and invariants exist.

### E3 — Teaching

For unfamiliar domains or deliberate learning. Build a mental model from current behavior to proposed behavior, use one or two concrete request examples, and explain consequences of alternatives. Keep all teaching tied to this design.

## Selection Rules

- Choose the highest engineering level triggered by a material part of the requested scope.
- Choose explanation depth from the reader's needs, not the engineering level. `Level 1 / E3` and `Level 3 / E1` are valid.
- Increase depth for irreversible migrations, privacy or security exposure, costly model use, weak observability, or uncertain repository evidence.
- Reduce depth when the repository already contains an authoritative decision record; link it and explain only the change-specific consequences.
- If scope is ambiguous between levels, state the assumption that determines the choice.

## Expected Size, Not a Quota

Depth controls coverage, not word count. A design is complete when a reviewer can explain the system change, decide whether to approve it, identify unresolved risks, and locate the implementation boundaries. Delete sections that do not help those outcomes.

