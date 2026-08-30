---
name: execution-plan
description: Turn a reviewed and approved implementation design into a repository-grounded coding-agent execution plan with exact targets, dependency-ordered tasks, tests, verification, and done criteria. Use after architecture and rationale are settled; do not use to invent or redesign the solution.
---

# Execution Plan

Produce the shortest plan that lets a coding agent implement the approved design correctly and verify the result. The design owns intent and architecture; this plan owns execution precision.

## Preconditions

- Identify the approved design, decision record, or explicitly approved technical direction.
- Never infer approval from a draft. If approval is absent, report that prerequisite instead of silently treating the design as final.
- Inspect the live repository before planning. The plan must reflect current paths, symbols, conventions, tests, commands, and local instructions.

## Workflow

1. Read the approved design and extract its outcomes, invariants, constraints, contracts, rollout requirements, and explicit exclusions.
2. Inspect every affected repository area and its closest tests. Trace call sites and dependencies far enough to name real edit targets.
3. Compare the design with repository reality. Surface missing abstractions, renamed paths, incompatible contracts, or unsafe assumptions in a conflict section. Do not silently change architecture.
4. Read [references/execution-plan-template.md](references/execution-plan-template.md) and use only the sections that help execution.
5. Read the example matching the scope when useful:
   - [small change](references/examples/small-change.md)
   - [medium change](references/examples/medium-change.md)
   - [architectural change](references/examples/architectural-change.md)
6. Order tasks by dependency and integration risk. Each task should produce one checkable result and name exact files, symbols, behavior, tests, verification commands, expected signals, and done criteria.
7. Include concise code, signatures, schemas, or pseudocode only where they remove ambiguity. Reuse repository conventions; do not paste routine boilerplate.
8. Save the plan to `docs/plans/execution/YYYY-MM-DD-<feature-name>.md`, using a lowercase kebab-case feature name.

## Planning Rules

- Prefer vertical, testable increments over batches such as “write all models, then all services, then all tests,” unless a migration dependency requires that order.
- For behavior changes, plan a failing test or other observable baseline first when practical, then the minimum implementation, then passing verification.
- Use exact commands accepted by the repository and state the expected pass or failure signal. Never invent a command when repository evidence is missing.
- Separate automated checks from human, staging, evaluation, migration, or rollout checks.
- Carry forward design requirements without reteaching their rationale. Link the approved design instead.
- Do not add adjacent refactors, dependencies, abstractions, or product behavior merely because they seem helpful.
- Make completion objective: every task and the whole plan must have observable done criteria.

## Conflict Policy

When repository reality conflicts with the approved design:

- record the observed evidence and the affected design decision;
- state whether execution is blocked or a compatible implementation choice exists inside the approved boundaries;
- continue planning unaffected work when useful;
- request a design decision before prescribing an architectural change.

Do not hide a conflict inside a task, reinterpret the design, or present a redesign as an implementation detail.

## Boundary With Design

Do not repeat background tutorials, broad alternatives, or architectural rationale. If important design information is missing, send the work back to design review instead of expanding the execution plan into a second design document.

