# Coding-Agent Planning Skills

This package contains two complementary, repository-agnostic Codex skills:

```text
Requirement or change request
          |
          v
implementation-design
Human-readable architecture, behavior, tradeoffs, and learning
          |
          v
Human review and approval
          |
          v
execution-plan
Repository-grounded, dependency-ordered implementation instructions
          |
          v
Coding and verification
```

## Bundles

- `implementation-design/` produces design plans at `docs/plans/YYYY-MM-DD-<feature-name>.md`.
- `execution-plan/` converts an approved design into an executable plan at `docs/plans/execution/YYYY-MM-DD-<feature-name>.md`.

Each folder is self-contained and can be installed as an individual skill. The examples are illustrative: their paths and symbols belong to hypothetical repositories and must never be copied into a real plan without inspecting that repository.

