# Example: Asynchronous Multimodal Ingestion — Implementation Design

> This is an illustrative Level 3 plan. Paths and symbols belong to a hypothetical repository.

- **Status:** Draft
- **Date:** 2026-08-30
- **Engineering depth:** Level 3
- **Explanation depth:** E2

## Executive Summary

Document upload currently performs text extraction, image understanding, chunking, embedding, and indexing inside one API request. Large multimodal documents exceed request timeouts and leave partially indexed data that is difficult to diagnose. Move ingestion to a durable asynchronous job with explicit stage state, idempotent workers, versioned artifacts, and atomic publication of a completed index generation.

## Goals and Non-Goals

### Goals

- Acknowledge uploads quickly and expose durable progress.
- Process text, tables, and images through specialized stages that can retry safely.
- Prevent partially processed document generations from appearing in retrieval.
- Make artifacts, model versions, failures, cost, and lineage traceable.
- Support cancellation, retry from a failed stage, and rollback to the last published generation.

### Non-Goals

- Replacing the existing vector database or retrieval API.
- Building a general-purpose workflow platform.
- Automatically choosing new extraction or embedding models.
- Reprocessing all historical documents during the first release.

## Current System

Observed flow:

- `src/api/documents.py::upload_document` stores the source and calls `IngestionService.ingest` synchronously.
- `src/ingestion/service.py` coordinates parser, vision model, chunker, embedder, and `VectorIndex.upsert` in process.
- `documents.index_status` is `pending`, `ready`, or `failed`, but does not identify stages or generations.
- Vector points are written incrementally under `document_id`; retrieval filters only by tenant and document authorization.
- Request cancellation can stop orchestration after some points are already visible.

```text
Upload request
   -> store source
   -> parse text/tables
   -> describe images
   -> chunk
   -> embed
   -> upsert visible points
   -> mark ready
```

The API request owns both user interaction and a long-running data pipeline. Incremental visibility means `failed` does not imply “no new data was published.”

## Proposed Architecture

Introduce a durable `IngestionJob` and stage queue. The API creates a document generation and job in one database transaction, then returns `202 Accepted`. Workers claim stages and write immutable, versioned artifacts. A publisher makes the generation retrievable only after validation succeeds.

```text
Client
  |
  v
Documents API -----> source object store
  |                         |
  +--> job + generation ----+
              |
              v
       durable stage queue
              |
   +----------+-----------+-----------+-----------+
   v                      v           v           v
extract worker       vision worker  chunker   embed/index worker
   |                      |           |           |
   +------ versioned artifact manifest -----------+
                              |
                              v
                       validation/publish
                              |
                 atomically set published_generation
                              |
                              v
                           Retrieval
```

### Ownership

- **Documents API:** authorization, source registration, job creation, status and cancellation endpoints.
- **Job store:** authoritative job, stage, attempt, lease, error, and generation state.
- **Object store:** immutable source and intermediate artifacts addressed by tenant/document/generation/stage/version.
- **Stage workers:** one transformation each; no worker publishes a generation.
- **Publisher:** validates manifest completeness and atomically changes the document's visible generation.
- **Retrieval:** reads only points matching `documents.published_generation`.

## State Model

```text
queued -> running -> validating -> published
   |        |             |
   |        +-> failed <--+
   |              |
   +-> cancelled  +-> queued (explicit retry from safe stage)
```

Terminal states are `published` and `cancelled`. `failed` retains retryable state but does not advance automatically after the retry budget is exhausted.

Each stage attempt uses a lease. An expired lease permits another worker to retry. Completion is accepted only if the worker still owns the lease, preventing a late worker from overwriting a newer attempt.

## Data Contracts

```python
IngestionJob(
    id: UUID,
    tenant_id: UUID,
    document_id: UUID,
    generation: int,
    status: JobStatus,
    requested_at: datetime,
    cancelled_at: datetime | None,
)

StageRun(
    job_id: UUID,
    stage: StageName,
    status: StageStatus,
    attempt: int,
    lease_owner: str | None,
    lease_expires_at: datetime | None,
    input_manifest_uri: str,
    output_manifest_uri: str | None,
    implementation_version: str,
    error_code: str | None,
)
```

Stage messages contain identifiers and manifest locations, not document content. At-least-once delivery is assumed. The idempotency key is `(job_id, stage, implementation_version)`; output is immutable, and committing stage completion is conditional on the active lease.

## Runtime Flows

### Successful ingestion

1. The API authorizes the tenant, stores the source, and transactionally creates generation `N` plus its job.
2. The dispatcher enqueues the first eligible stage using an outbox record committed with the job.
3. Workers transform versioned inputs into versioned outputs and commit manifests.
4. The coordinator releases dependent stages only after their prerequisites succeed.
5. The embedding/index worker writes points tagged with generation `N`; they remain invisible to retrieval.
6. Validation checks artifact completeness, point counts, tenant metadata, and sampled referential integrity.
7. The publisher atomically sets `documents.published_generation = N` and marks the job published.
8. Older generations remain available for the rollback window, then are garbage-collected by policy.

### Retry after a model timeout

The vision stage records a typed transient error and retries with bounded exponential backoff. Because its output path is versioned by stage implementation, a repeated attempt either reuses a committed valid artifact or writes the same logical output before conditionally completing the stage. After the retry budget, the job enters `failed`; an operator or user may explicitly retry it.

### Cancellation

Cancellation marks the job and prevents new stages from being released. A running model call may finish, but its completion cannot publish the generation. Immutable artifacts are later cleaned by retention policy.

## Design Decisions

### Durable job state rather than queue state as authority

Queues deliver work; they do not provide the complete user-visible history, dependencies, or publication decision. The relational job store remains the authority for state transitions and leases.

### At-least-once delivery with idempotent stages

Exactly-once processing across a queue, object store, model providers, database, and vector index is not realistic. Idempotent outputs and conditional state transitions make duplicate delivery safe.

### Generation-based atomic publication

Index writes are not assumed to be transactionally atomic. Tagging every point with a generation and changing one relational pointer ensures retrieval sees the old complete generation or the new complete generation, never an in-progress mixture.

### Version intermediate artifacts

Model prompts, parsers, and embedding implementations change. Recording implementation versions enables lineage, targeted reprocessing, reproducible evaluation, and safe coexistence during rollout.

## Security, Privacy, and Safety

- Every job, message, artifact path, and vector point carries tenant identity; workers validate it against job state rather than trusting the message alone.
- Workers receive least-privilege access for their stage and deployment region.
- Source and artifacts use encryption and the document retention policy; raw content is excluded from logs and queue messages.
- Extracted document instructions are treated as untrusted content. Vision and extraction stages cannot call tools or modify job control state.
- Model provider selection must comply with tenant residency and data-processing policy.
- Status APIs expose typed, sanitized error codes; provider payloads remain restricted operational data.

## Observability and Evaluation

Operational signals include queue age, stage duration, attempt counts, lease expirations, failure codes, artifact sizes, provider latency and cost, index point counts, publication lag, cancellations, and generation cleanup backlog. Traces propagate `job_id`, `document_id`, `generation`, and `stage_run_id` without content.

Quality gates use a versioned multimodal corpus to measure:

- text and table extraction accuracy;
- image-description factuality and coverage;
- chunk boundary quality;
- retrieval recall@k by modality;
- citation correctness in downstream answers;
- cross-version regression by parser, prompt, model, and embedding version.

Publication validation protects structural integrity. Offline evaluation authorizes implementation-version rollout; it does not run in every job.

## Capacity and Cost

Workers scale independently by stage. The coordinator applies per-tenant concurrency and global provider budgets. Backpressure appears as queue age, not API timeouts. Large-document limits are enforced before job creation, and estimated modality counts support early cost controls.

## Implementation Mapping

| Area | Repository location | Responsibility change |
|---|---|---|
| API | `src/api/documents.py` | Create jobs; expose status, retry, and cancellation |
| Domain | `src/ingestion/domain.py` | Job, stage, generation, and transition rules |
| Persistence | `src/ingestion/repository.py`; `migrations/` | Durable state, leases, outbox, published pointer |
| Coordination | `src/ingestion/coordinator.py` | Release dependency-ready stages |
| Workers | `src/workers/ingestion/` | Idempotent stage implementations |
| Artifacts | `src/ingestion/artifacts.py` | Versioned manifest and object naming |
| Indexing | `src/retrieval/vector_index.py` | Generation-tagged writes and reads |
| Publication | `src/ingestion/publisher.py` | Validate and atomically publish |
| Operations | `deploy/workers/`; `dashboards/ingestion/` | Independent scaling, alerts, and budgets |
| Evaluation | `evals/multimodal_ingestion/` | Cross-version quality gates |

## Migration and Rollout

1. Add job/generation schema and generation-aware retrieval while existing documents map to generation `1`.
2. Deploy workers and run shadow ingestions that never publish; compare artifacts and evaluation results.
3. Enable asynchronous ingestion for an internal tenant behind a flag.
4. Expand by tenant while monitoring publication lag, failures, duplicate attempts, quality, and cost.
5. Stop synchronous ingestion after rollback and operational readiness criteria are met.
6. Retain the previous published generation for the rollback window; garbage collection requires confirmed non-publication and age.

## Alternatives Considered

### Increase the API timeout

Rejected because it does not solve partial visibility, retry recovery, independent scaling, lineage, or cancellation.

### One monolithic background worker

Useful as an intermediate migration, but rejected as the target because extraction, vision, and embedding have different resource, provider, retry, and scaling characteristics. The durable stage model still permits co-locating workers initially.

### Publish points incrementally

Rejected because users could retrieve incomplete or internally inconsistent document content. Generation publication provides a simpler visible-state invariant.

## Risks and Mitigations

| Risk | Consequence | Mitigation or decision needed |
|---|---|---|
| Duplicate delivery creates conflicting work | Cost or corrupt stage state | Idempotency keys, immutable artifacts, conditional leases |
| Orphaned generations accumulate | Storage and index growth | Retention policy and measured garbage collector |
| Schema and worker versions drift | Unreadable jobs or artifacts | Versioned contracts and compatibility window |
| Tenant data crosses boundaries | Severe privacy incident | Tenant validation at every boundary and least privilege |
| New path silently lowers retrieval quality | Poor answers despite healthy jobs | Versioned evaluation gate and modality metrics |

## Open Questions

- Which queue technology and transactional-outbox mechanism fit the repository's deployed infrastructure?
- What publication rollback window satisfies both storage cost and incident recovery requirements?
- Which stage failures may be retried by end users versus operators only?
- What evaluation thresholds authorize a new parser, prompt, vision model, or embedding version?

