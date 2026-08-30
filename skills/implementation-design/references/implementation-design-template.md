# Implementation-Design Template

Adapt this template to the selected depth. Remove irrelevant sections and add domain-specific ones when they support a design decision.

```markdown
# {Feature name} — Implementation Design

- **Status:** Draft | In review | Approved
- **Date:** YYYY-MM-DD
- **Engineering depth:** Level 1 | Level 2 | Level 3
- **Explanation depth:** E1 | E2 | E3
- **Owners / reviewers:** {Only when known}
- **Related requirement:** {Issue, brief, or concise request}

## Executive Summary

State the problem, the proposed shape of the solution, and the most important consequence in a short paragraph.

## Goals and Non-Goals

### Goals

- Observable outcome.

### Non-Goals

- Adjacent work intentionally excluded.

## Constraints and Assumptions

- Confirmed constraint, with repository evidence where available.
- Assumption that requires review.

## Current System

Describe only the current behavior needed to understand the change. Cite repository paths, symbols, contracts, and configuration.

```text
Current runtime or data flow
```

## Proposed Design

Explain component responsibilities and the invariants the design preserves.

```text
Proposed runtime or data flow
```

### Request and Data Flow

Walk through one representative success path. Add failure or recovery flows when they materially differ.

### Interfaces and Data Contracts

Describe meaningful input, output, event, state, configuration, or schema changes. Use concise pseudocode or schemas when precision helps.

### AI-System Behavior

When relevant, cover:

- prompt and model responsibilities;
- retrieval, ranking, context construction, and citations;
- tool permissions and trust boundaries;
- session, memory, and workflow state;
- deterministic validation and fallback behavior;
- data sensitivity, retention, and lineage;
- latency, token or inference cost, and capacity;
- offline evaluation, online signals, and regression thresholds.

## Design Decisions

For each material decision, state the choice, why it fits this system, and its consequence. Discuss alternatives only when they were plausible.

## Failure Modes and Recovery

Describe failures that change user-visible behavior, data integrity, cost, or operations. State retry, timeout, degradation, replay, or human-intervention behavior as applicable.

## Security, Privacy, and Safety

Include only relevant trust boundaries, authorization, tenant isolation, prompt-injection handling, sensitive-data policy, auditability, or content-safety considerations.

## Observability and Evaluation

Define the signals needed to distinguish healthy behavior from silent AI-quality regression. Include operational telemetry and outcome-quality evaluation as applicable.

## Implementation Mapping

Map responsibilities to existing or proposed modules and symbols. Keep this at component or change-set granularity rather than task-by-task execution order.

| Area | Repository location | Responsibility change |
|---|---|---|
| {Area} | `{path or proposed path}` | {Design-level mapping} |

## Rollout and Migration

Describe compatibility, feature flags, backfill, staged rollout, rollback, and deprecation only when relevant.

## Verification Strategy

Explain what unit, integration, end-to-end, evaluation, load, or failure-injection evidence will demonstrate that the design works. Reserve exact commands for the execution plan unless a command itself is a design constraint.

## Risks and Mitigations

| Risk | Consequence | Mitigation or decision needed |
|---|---|---|
| {Risk} | {Impact} | {Response} |

## Open Questions

- Decision that remains unresolved, its owner if known, and what it blocks.

## Review Checklist

- [ ] Goals, non-goals, and assumptions are agreed.
- [ ] Runtime and data flows are understandable.
- [ ] Contracts, failure behavior, and AI-quality evaluation are sufficient.
- [ ] Security, operational, migration, and cost concerns are addressed where relevant.
- [ ] Repository mapping is credible.
- [ ] Open questions are resolved or explicitly accepted.
```

After approval, retain this document as the architectural source for the corresponding execution plan. Do not replace it with the execution plan.

