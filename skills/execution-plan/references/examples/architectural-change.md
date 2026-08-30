# Example: Asynchronous Multimodal Ingestion — Execution Plan

> Illustrative repository only. Its paths, symbols, and commands are examples of the precision expected after real inspection.

- **Date:** 2026-08-30
- **Status:** Ready
- **Approved design:** `docs/plans/2026-08-30-async-multimodal-ingestion.md`
- **Implementation scope:** Introduce durable staged ingestion and generation-based publication without changing the public retrieval request contract.

## Execution Contract

- The relational job store is authoritative; queue delivery is at least once.
- Stage completion requires an active lease and idempotency key `(job_id, stage, implementation_version)`.
- Intermediate artifacts are immutable and versioned.
- Retrieval exposes only `documents.published_generation`.
- API, messages, logs, and vector metadata preserve tenant isolation; queue messages contain no document content.
- Existing synchronous ingestion remains available behind its current flag until rollout exit criteria are met.

## Repository Findings

- PostgreSQL migrations use Alembic under `migrations/versions/` and domain repositories use SQLAlchemy transactions.
- `src/api/documents.py::upload_document` already stores source objects before invoking `IngestionService.ingest`.
- `src/queue/outbox.py::OutboxPublisher` provides post-commit dispatch and retry.
- `src/retrieval/vector_index.py` supports metadata filters but currently writes only `tenant_id` and `document_id`.
- Worker deployments are defined in `deploy/kustomize/workers/`; dashboards and alerts live under `observability/`.
- Repository checks run with `make test`, `make integration-test`, and `make lint`.

## Design Conflicts or Blockers

None found. The existing outbox and vector metadata filters satisfy the approved integration assumptions.

## Dependency Order

```text
Task 1 schema/state model
   +--> Task 2 repository + transactional outbox --> Task 6 API/status
   +--> Task 3 artifact manifests ------------+
   +--> Task 4 leases + stage execution -------+--> Task 7 coordinator/publisher
   +--> Task 5 generation-aware vector index --+
                                                 +--> Task 8 deployment, evaluation, rollout
```

## Task 1 — Durable jobs, stages, and document generations exist

- **Depends on:** None
- **Files:**
  - Create: `migrations/versions/20260830_add_ingestion_jobs.py`
  - Create: `src/ingestion/domain.py`
  - Modify: `src/db/models/document.py`
  - Create: `src/db/models/ingestion_job.py`
  - Create: `tests/ingestion/test_domain.py`
  - Create: `tests/migrations/test_ingestion_jobs.py`
- **Symbols:** `IngestionJob`, `StageRun`, `JobStatus`, `StageStatus`, `transition_job`, `Document.published_generation`

### Change

Add the approved job, stage-attempt, generation, lease, error-code, manifest, and implementation-version fields. Add uniqueness for `(job_id, stage, implementation_version)` and indexes for runnable stages and document status. Backfill existing ready documents to published generation `1`; non-ready documents retain no published generation.

Encode allowed state transitions as pure domain functions and reject invalid or terminal-state transitions.

### Verification

1. Add domain tests for every allowed transition plus stale, cancelled, and terminal rejections.
2. Add migration tests for upgrade, existing-document backfill, constraints, indexes, and downgrade behavior supported by repository policy.
3. Run `pytest tests/ingestion/test_domain.py tests/migrations/test_ingestion_jobs.py -q`.

### Done When

- [ ] The database represents every approved job and stage state.
- [ ] Existing ready documents remain retrievable as generation `1` after migration.
- [ ] Invalid transitions fail before persistence.

## Task 2 — Job creation and first-stage dispatch are transactionally coupled

- **Depends on:** Task 1
- **Files:**
  - Create: `src/ingestion/repository.py`
  - Modify: `src/queue/outbox.py`
  - Create: `tests/ingestion/test_repository.py`
  - Modify: `tests/queue/test_outbox.py`
- **Symbols:** `IngestionJobRepository.create`, `claim_stage`, `complete_stage`, `fail_stage`, `OutboxPublisher`

### Change

Implement repository operations with compare-and-set conditions for state and lease ownership. Create generation, job, first stage, and outbox record in one transaction. Outbox payloads carry IDs, stage name, implementation version, and manifest URI only.

### Verification

1. Prove rollback leaves neither job nor outbox record.
2. Prove duplicate dispatch creates no duplicate stage identity.
3. Prove stale lease owners cannot complete or fail a newer attempt.
4. Run `pytest tests/ingestion/test_repository.py tests/queue/test_outbox.py -q` with the repository's PostgreSQL test fixture.

### Done When

- [ ] A committed job always has dispatchable work, and rolled-back jobs have none.
- [ ] Lease and idempotency invariants hold under duplicate and late operations.

## Task 3 — Every stage reads and writes a versioned immutable manifest

- **Depends on:** Task 1
- **Files:**
  - Create: `src/ingestion/artifacts.py`
  - Create: `src/ingestion/schemas/artifact_manifest.py`
  - Create: `tests/ingestion/test_artifacts.py`
- **Symbols:** `ArtifactManifest`, `ArtifactStore.read`, `write_once`, `artifact_uri`

### Change

Implement tenant/document/generation/stage/implementation-version object keys and a schema containing input lineage, output objects, content hashes, counts, model or prompt version, and creation metadata. `write_once` accepts byte-identical retries and rejects conflicting content at an existing URI.

### Verification

1. Cover deterministic URI generation, schema validation, content-hash verification, identical retry, conflicting retry, and tenant mismatch.
2. Run `pytest tests/ingestion/test_artifacts.py -q` against the existing object-store fake.

### Done When

- [ ] A downstream stage can verify all inputs and lineage without reading job-control data from a queue message.
- [ ] Conflicting writes cannot replace committed artifacts.

## Task 4 — Workers execute one leased, idempotent stage at a time

- **Depends on:** Tasks 2 and 3
- **Files:**
  - Create: `src/workers/ingestion/base.py`
  - Create: `src/workers/ingestion/extract.py`
  - Create: `src/workers/ingestion/vision.py`
  - Create: `src/workers/ingestion/chunk.py`
  - Create: `src/workers/ingestion/embed.py`
  - Create: `tests/workers/ingestion/test_stage_runner.py`
- **Symbols:** `StageRunner.handle`, `StageHandler.run`, stage handler classes

### Change

Build a common runner that validates message tenant identity against the job, claims or renews a lease, reads the input manifest, invokes exactly one handler, commits its output manifest, and conditionally completes the stage. Map provider timeouts and rate limits to retryable codes; map validation and policy failures to terminal stage codes. Respect cancellation before work and before completion.

### Verification

1. Contract-test every handler with the same runner fixture.
2. Cover duplicate delivery, expired lease takeover, late completion, transient retry budget, terminal error, cancellation, and redacted log fields.
3. Run `pytest tests/workers/ingestion -q`.

### Done When

- [ ] Duplicate or late workers cannot corrupt state or replace artifacts.
- [ ] Typed error and cancellation behavior matches the approved state model.
- [ ] Logs and messages contain no source or extracted content.

## Task 5 — Vector indexing and retrieval are generation-aware

- **Depends on:** Task 1
- **Files:**
  - Modify: `src/retrieval/vector_index.py`
  - Modify: `src/retrieval/retriever.py`
  - Create: `tests/retrieval/test_document_generations.py`
- **Symbols:** `VectorIndex.upsert_chunks`, `VectorRetriever.search`, `Document.published_generation`

### Change

Require tenant, document, and generation metadata on ingestion writes. Join or preload each authorized document's published generation and include it in retrieval filters. Add delete-by-generation for retention cleanup; do not call it from request handling.

### Verification

1. Index old published, new unpublished, and new published generations for one document.
2. Assert retrieval sees only the relationally published generation before and after atomic pointer change.
3. Assert cross-tenant and unspecified-generation queries cannot return points.
4. Run `pytest tests/retrieval/test_document_generations.py -q` and the existing retrieval integration suite.

### Done When

- [ ] Partial and unpublished points are invisible.
- [ ] Publication changes visibility without rewriting points.
- [ ] Tenant filters remain mandatory.

## Task 6 — Upload, status, retry, and cancellation expose durable job behavior

- **Depends on:** Task 2
- **Files:**
  - Modify: `src/api/documents.py`
  - Create: `src/api/schemas/ingestion.py`
  - Modify: `tests/api/test_documents.py`
- **Symbols:** `upload_document`, `get_ingestion_status`, `retry_ingestion`, `cancel_ingestion`

### Change

Behind `async_ingestion_enabled`, replace the synchronous service call with transactional job creation and return `202` plus job and status resource identifiers. Add authorized status, explicit retry, and cancellation endpoints using sanitized error codes. Retain the existing synchronous branch unchanged while the flag is off.

### Verification

1. Cover flag-off compatibility; flag-on `202`; authorization; status transitions; invalid retry; cancellation; and sanitized failures.
2. Assert source storage failure creates no job and job transaction failure does not dispatch work.
3. Run `pytest tests/api/test_documents.py -q`.

### Done When

- [ ] Clients can observe and control the approved job lifecycle.
- [ ] The flag-off path has unchanged response behavior.
- [ ] API payloads expose neither provider errors nor internal artifact locations.

## Task 7 — The coordinator releases dependencies and publishes only validated generations

- **Depends on:** Tasks 2, 3, 4, and 5
- **Files:**
  - Create: `src/ingestion/coordinator.py`
  - Create: `src/ingestion/publisher.py`
  - Create: `tests/ingestion/test_coordinator.py`
  - Create: `tests/ingestion/test_publisher.py`
- **Symbols:** `IngestionCoordinator.on_stage_completed`, `GenerationPublisher.validate`, `publish`

### Change

Release a stage only when all approved prerequisites are complete and the job is neither failed nor cancelled. Before publication, validate manifest completeness, expected artifact and point counts, tenant metadata, and sampled referential integrity. In one database transaction, compare the job's generation and set `published_generation`, then mark the job `published`.

### Verification

1. Cover dependency fan-in, duplicate completion events, cancelled jobs, failed prerequisites, incomplete manifests, count mismatches, tenant mismatch, concurrent publication, and successful atomic publication.
2. Add an integration test that kills coordination after index writes but before publication; retrieval must continue to serve the old generation.
3. Run `pytest tests/ingestion/test_coordinator.py tests/ingestion/test_publisher.py -q`, then `make integration-test` for the ingestion target.

### Done When

- [ ] No stage runs before its dependencies.
- [ ] Only structurally valid generations publish.
- [ ] A crash before publication leaves the old complete generation visible.

## Task 8 — Workers are deployable, observable, evaluable, and reversible

- **Depends on:** Tasks 4, 6, and 7
- **Files:**
  - Create: `deploy/kustomize/workers/ingestion/`
  - Create: `observability/dashboards/ingestion.json`
  - Create: `observability/alerts/ingestion.yaml`
  - Create: `evals/multimodal_ingestion/runner.py`
  - Create: `evals/multimodal_ingestion/cases.jsonl`
  - Modify: `docs/operations/ingestion.md`
- **Symbols:** worker entry points, `MultimodalIngestionEvaluator`

### Change

Define independently scalable worker deployments with least-privilege service identities, queue and provider budgets, and region-compatible configuration. Add approved job, stage, lease, queue-age, failure, artifact, point-count, publication-lag, cleanup, provider-latency, and cost signals. Add the versioned quality evaluator and document flag enablement, cohort expansion, rollback to the prior generation, cancellation, manual retry, and safe generation cleanup.

### Verification

1. Run `make lint` and repository deployment validation.
2. Run `python -m evals.multimodal_ingestion --dataset evals/multimodal_ingestion/cases.jsonl`; expect extraction, modality coverage, retrieval recall, and citation metrics split by implementation version.
3. In staging, inject duplicate messages, provider timeout, worker termination, stale lease, cancellation, and publisher failure; confirm the dashboard, alerts, recovery behavior, and old-generation visibility.
4. Rehearse flag disablement and published-generation rollback without deleting either generation.

### Done When

- [ ] Each stage can scale and fail independently within approved budgets.
- [ ] Operators can locate a job's stage, attempt, lineage, and sanitized failure from IDs.
- [ ] Quality gates and failure-injection scenarios produce recorded evidence.
- [ ] Rollback and cleanup procedures are executable and reviewed.

## Final Verification Matrix

| Outcome or risk | Evidence | Command / environment | Expected signal |
|---|---|---|---|
| Durable, valid state transitions | Domain, migration, repository tests | `pytest tests/ingestion tests/migrations -q` | All transition and lease cases pass |
| At-least-once safety | Worker and coordinator tests | `pytest tests/workers/ingestion tests/ingestion/test_coordinator.py -q` | Duplicates and stale attempts are harmless |
| Atomic visible generation | Retrieval and publisher integration tests | `make integration-test` | Only one complete generation is visible |
| Tenant isolation and content-free control plane | API, worker, retrieval assertions | `make test` | Cross-tenant access denied; payload checks pass |
| Multimodal quality | Versioned evaluation report | `python -m evals.multimodal_ingestion --dataset evals/multimodal_ingestion/cases.jsonl` | Meets approved thresholds |
| Recovery and rollback | Staging failure-injection record | Staging | Jobs recover or stop as designed; old generation remains available |

## Non-Automated Checks

- Security review of worker identities, tenant validation, provider residency, retention, and error exposure.
- Capacity review of queue age, provider quotas, large-document limits, and per-tenant concurrency.
- Staged rollout approval after shadow artifact comparison and multimodal evaluation.
- Rollback and garbage-collection rehearsal before disabling synchronous ingestion.

## Completion Criteria

- [ ] All eight tasks meet their done criteria.
- [ ] `make test`, `make integration-test`, `make lint`, migration checks, and deployment validation pass.
- [ ] Offline quality, staging failure-injection, security, capacity, and rollback evidence meets the approved design gates.
- [ ] The asynchronous path remains behind its flag until rollout approval.
- [ ] No unresolved conflict or inferred architecture change was implemented.

