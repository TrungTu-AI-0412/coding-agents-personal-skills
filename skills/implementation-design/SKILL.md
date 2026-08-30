---
name: implementation-design
description: Create human-facing implementation designs that explain a proposed software change, its architecture, runtime and data flows, decisions, risks, and repository mapping before coding. Use when a developer wants to understand, review, or manually implement a change; do not use for a terse agent execution checklist.
---

# Implementation Design

Produce a technically grounded design that a human can review, learn from, and implement manually. This skill stops at design unless the user separately asks for implementation.

## Workflow

1. Inspect the repository, local instructions, relevant code, tests, configuration, and documentation. Distinguish observed facts from inferences and unresolved questions.
2. Read [references/plan-depth-policy.md](references/plan-depth-policy.md) and select the smallest design and explanation depth that fits the change.
3. Read [references/implementation-design-template.md](references/implementation-design-template.md) and [references/developer-learning-guidance.md](references/developer-learning-guidance.md). Adapt the structure; do not fill sections mechanically.
4. Read only the matching example when useful:
   - [small change](references/examples/small-change.md)
   - [medium change](references/examples/medium-change.md)
   - [architectural change](references/examples/architectural-change.md)
5. Explain the current behavior, proposed behavior, runtime and data flow, affected contracts, design decisions, implementation mapping, verification strategy, risks, and open questions. Include alternatives only when they clarify a real decision.
6. Save the design to `docs/plans/YYYY-MM-DD-<feature-name>.md`, using a lowercase kebab-case feature name. Mark it `Draft`, `In review`, or `Approved`; never infer approval.

## Required Qualities

- Ground claims in repository evidence and cite paths and symbols when available.
- Preserve the user's requested scope and separate goals from non-goals.
- Make AI-system behavior explicit: model boundaries, prompts, retrieval, tool calls, state, data lineage, fallbacks, evaluation, safety, latency, and cost where relevant.
- Teach through the actual change, not through generic tutorials.
- Provide enough implementation mapping for manual coding without turning the document into an agent task ledger.
- Surface uncertainty and design conflicts instead of presenting guesses as decisions.

## Boundary With Execution Planning

Do not decompose the work into tiny sequential coding steps, prescribe routine commands, or repeat every expected edit. After the design is reviewed and approved, use the separate `execution-plan` skill to produce that artifact.

