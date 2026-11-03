# Implementation Plan: LLM Observability Platform

## Overview

This plan builds the MVP incrementally and test-first across three components and the local deployment, proving a vertical slice (SDK → ingestion → database) before fanning out to the query API, dashboard, and deployment:

- **Backend** (Python 3.12, FastAPI, SQLAlchemy 2.x, PostgreSQL) — pure derivation services (cost, trace status/latency), SQLAlchemy models, atomic/idempotent ingestion, authentication + project scoping, the query API with filtering and keyset pagination, schema-on-startup, and the Admin CLI.
- **SDK** (Python 3.12, `httpx`) — `contextvars` attribution, span recording, a bounded drop-oldest queue, a background batching sender with retry/suppression, and the public `init`/`observe`/`wrap_llm`/`flush` surface.
- **Dashboard** (React + TypeScript) — the pure `buildSpanTree` reconstruction function, the API-key gate, the trace list with filters + pagination, and the trace-detail span-tree view.
- **Deployment** — Docker Compose wiring for Backend, Database, and Dashboard, plus smoke/integration checks.

The order first lands the ingestion vertical slice: scaffold, then cost and derivation (prerequisites of ingestion), then models/schema/test-fixture, auth, ingest validation, and the ingestion endpoint. The SDK follows so an end-to-end check can prove a trace recorded via the SDK is persisted and retrievable. Only then do the query API, Admin CLI, dashboard, and Docker Compose follow, with CI and a demo near the end and the full verification gate last. Each correctness property (1–25) is implemented as a single property-based test (Hypothesis for Python, fast-check for TypeScript) placed close to the code it verifies. ADRs are recorded under `docs/adr/` in the same tasks that implement the decisions they describe.

## Tasks

- [x] 1. Scaffold repository structure and tooling
  - Create the top-level layout: `backend/` (FastAPI app, models, services, CLI), `sdk/observability/` (Python SDK package), `dashboard/` (React + TypeScript app), `docs/adr/`, and `docker/` for Compose assets.
  - Add `backend/pyproject.toml` and `sdk/pyproject.toml` declaring Python 3.12, FastAPI, SQLAlchemy 2.x, `psycopg`/driver, `httpx`, `pydantic`, `pytest`, and `hypothesis`; add shared pytest configuration.
  - Initialize the dashboard with Vite (React + TypeScript), and add `vitest`, React Testing Library, and `fast-check` to `dashboard/package.json`.
  - Add placeholder package `__init__.py` files and a root README section describing the component layout.
  - Create `docs/adr/` with a single ADR template file (`0000-template.md`) containing the sections Context / Decision / Consequences / Alternatives considered. Create the seven ADRs (`0001-sdk-generated-ids.md` through `0007-atomic-idempotent-ingest.md`) as **"Status: Draft" STUB files** — each with only its title, status, and placeholder sections — to be rewritten in the maintainer's own words. Each stub is finalized (expanded into real Context / Decision / Consequences / Alternatives) in the same change set as the decision it describes.
  - _Requirements: 12.8_

- [ ] 2. Implement the backend cost service
  - [ ] 2.1 Implement `backend/cost.py` with decimal arithmetic (manual - implemented by the maintainer, not by an agent)
    - Implement `ModelPrice`, `PriceTable` (with `get`), `CostResult`, and `compute_span_cost` using `decimal.Decimal` exactly as specified in the design.
    - Implement price-table loading from JSON/env configuration.
    - Return cost 0 / `price_unknown=False` when model or a token count is absent; cost 0 / `price_unknown=True` when model is present but not in the table; otherwise `prompt*prompt_price + completion*completion_price`.
    - Implement a `trace_cost` helper that sums span costs with `Decimal`.
    - _Requirements: 4.1, 4.2, 4.3, 4.4, 4.5, 3.7_
  - [ ]* 2.2 Write property test for span cost (Hypothesis)
    - **Property 4: Span cost is a total function with exact decimal arithmetic**
    - **Validates: Requirements 3.7, 4.1, 4.2, 4.3, 4.4**
  - [ ]* 2.3 Write property test for trace cost summation (Hypothesis)
    - **Property 5: Trace cost is the sum of span costs**
    - **Validates: Requirements 4.5**
  - [ ] 2.4 Write example/unit tests for cost computation
    - In-table model with a known input/output token split produces the expected cost within 0.0001 currency units; a model absent from the table stores cost zero and sets the unknown-price flag.
    - Mandatory: part of the Requirement 12 verification gate.
    - _Requirements: 12.3, 12.4_

- [ ] 3. Implement trace status and latency derivation
  - [ ] 3.1 Implement `backend/derivation.py` (manual - implemented by the maintainer, not by an agent)
    - Implement a pure function computing trace status (`error` if any span has status `error`, else `success`) and trace latency (max `end_time` minus min `start_time`) from a span collection.
    - _Requirements: 5.12_
  - [ ]* 3.2 Write property test for status/latency derivation (Hypothesis)
    - **Property 11: Trace status and latency derivation**
    - **Validates: Requirements 5.12**

- [ ] 4. Define SQLAlchemy data models, schema creation, and the test fixture
  - [ ] 4.1 Implement `backend/models.py` and database session setup (manual - implemented by the maintainer, not by an agent)
    - Define `Project`, `Trace`, and `Span` ORM models matching the design tables, including the composite `(trace_id, span_id)` primary key on spans, `NUMERIC(20,10)` cost, `price_unknown` boolean, JSONB input/output/metadata, CHECK constraints for `span_type`/`status`/`end_time >= start_time`, and the documented indexes.
    - Implement the database engine/session factory configured from environment.
    - _Requirements: 6.1, 6.6, 1.9, 3.4, 3.6, 3.10, 4.1, 4.3_
  - [ ] 4.2 Implement guarded schema creation on startup
    - Run `Base.metadata.create_all()` in a guarded startup step; on failure report the error, leave no partial schema, and keep `/healthz` not-ready.
    - Add `GET /healthz` returning ready only after schema creation succeeds.
    - _Requirements: 11.4, 11.5_
  - [ ]* 4.3 Write unit tests for models and schema-creation guard
    - Verify constraints are enforced and that a failed schema-creation step leaves `/healthz` not-ready with no partial schema.
    - _Requirements: 11.4, 11.5_
  - [ ] 4.4 Implement disposable PostgreSQL test fixture
    - Provision a disposable PostgreSQL (testcontainers or a dedicated Compose test service) with per-test isolation via transactional rollback or a fresh schema per test, exposed as a shared pytest fixture.
    - All property/example tests that touch the database depend on this fixture: 7.2, 7.3, 7.4, 7.5, the new end-to-end integration test (12.1), and the query-API DB tests 13.4–13.9.
    - Mandatory (not optional): every DB-touching test requires it.
    - _Requirements: 11.4, 12.1_

- [ ] 5. Implement authentication and project scoping
  - [ ] 5.1 Implement `backend/auth.py` key hashing and project resolution
    - Implement SHA-256 hashing over the raw key (with the `lop_` prefix convention) and a `require_project` FastAPI dependency that extracts `X-API-Key`, resolves the project by `api_key_hash`, and raises the correct `HTTPException`.
    - Missing key → 401 `missing_credentials`; no matching project → 401 `invalid_credentials` (non-disclosing).
    - _Requirements: 6.1, 6.2, 6.3, 6.4, 6.5_
  - [ ]* 5.2 Write property test for API-key resolution round-trip (Hypothesis)
    - **Property 12: API-key resolution round-trip**
    - **Validates: Requirements 3.2, 6.1, 6.4**
  - [ ]* 5.3 Write property test for missing/invalid credential rejection (Hypothesis)
    - **Property 14: Missing or invalid credentials are rejected without data**
    - **Validates: Requirements 3.3, 5.2, 6.3, 6.5**

- [ ] 6. Implement ingestion request DTOs and validation
  - [ ] 6.1 Implement Pydantic ingest models and field validation
    - Implement `IngestRequest`/`IngestSpan` enforcing SDK-generated `trace_id`/`span_id`, non-empty `name`, enum `span_type`, `status`, required `start_time`/`end_time` with `end_time >= start_time`, and the shared error envelope (`validation_error` with `span_index` + `field`).
    - Implement per-field size checks for `input`/`output` against the configurable max payload (default 65536 bytes), and span-count bounds (1..1000).
    - _Requirements: 3.1, 3.4, 3.5, 3.9, 3.11_
  - [ ]* 6.2 Write property test for atomic rejection of invalid spans (Hypothesis)
    - **Property 9: Invalid spans are rejected atomically and identified**
    - **Validates: Requirements 3.4, 3.5**
  - [ ]* 6.3 Write property test for malformed/oversize rejection (Hypothesis)
    - **Property 10: Malformed or oversize requests are rejected without persistence**
    - **Validates: Requirements 3.1, 3.9, 3.11**

- [ ] 7. Implement the ingestion endpoint with atomic, idempotent persistence
  - [ ] 7.1 Implement `POST /v1/ingest`
    - Wire the processing order: auth → parse/shape → per-field size → field validation → cost computation → atomic upsert-on-conflict-do-nothing keyed by `(trace_id, span_id)` → 2xx response with `spans_received`/`spans_persisted`/`spans_ignored_duplicate`.
    - Persist the trace row and spans in a single transaction preserving each `parent_span_id`; compute and store span cost/`price_unknown` at ingest.
    - When spans of one trace arrive across several ingest batches, maintain the trace row's derived `start_time`/`end_time` with a LEAST/GREATEST upsert: `start_time = LEAST(existing start_time, incoming min start_time)` and `end_time = GREATEST(existing end_time, incoming max end_time)`. Trace status stays correct across batches because it is derived from the persisted spans (or maintained monotonically so that any `error` span makes the trace status `error`, irrespective of batch arrival order).
    - _Requirements: 3.2, 3.6, 3.7, 3.8, 3.10, 3.11, 4.2, 4.3, 4.4_
  - [ ]* 7.2 Write property test for idempotency by (trace_id, span_id) (Hypothesis, test DB)
    - **Property 7: Ingestion is idempotent by (trace_id, span_id)**
    - **Validates: Requirements 3.10**
  - [ ]* 7.3 Write property test for atomic persistence preserving parents (Hypothesis, test DB)
    - **Property 8: Valid ingestion is atomic and preserves parent references**
    - **Validates: Requirements 3.6, 3.8**
  - [ ]* 7.4 Write property test for cost immutability across price-table updates (Hypothesis, test DB)
    - **Property 6: Stored cost is immutable across price-table updates**
    - **Validates: Requirements 4.6**
  - [ ] 7.5 Write example/unit tests for ingestion valid/invalid/idempotent paths
    - Valid payload persists the trace; missing-required-field and malformed-body payloads are rejected naming the failed field with nothing persisted; re-ingesting an existing span id creates no second copy and returns 2xx.
    - Mandatory: part of the Requirement 12 verification gate.
    - _Requirements: 12.1, 12.2, 12.6_
  - [ ] 7.6 Write a multi-batch trace ingestion test
    - Ingest a single trace in two separate batches, including an out-of-order batch where later spans arrive first; assert the final trace `start_time`/`end_time` are the LEAST/GREATEST across batches, that trace status is correct (error if any span errored regardless of batch order), and that span-tree reconstruction over the persisted spans is correct.
    - Mandatory: validates multi-batch trace-row maintenance.
    - _Requirements: 3.6, 3.10, 5.12_

- [ ] 8. Checkpoint - foundational backend services
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 9. Implement the SDK context and span recording
  - [ ] 9.1 Implement `contextvars` context and `SpanRecorder` (manual - implemented by the maintainer, not by an agent)
    - Implement `active_trace`/`active_span` context vars, span-id/trace-id generation (UUIDv4 strings), parent attribution (root when no active trace, else `parent_span_id` = active span id), restoration of the previous active span on exit, and `SpanData` capture of name/type/input/output/start/end/status; on exception set status `error`, capture the error, record end time, and re-raise.
    - Capture LLM `model`/`prompt_tokens`/`completion_tokens` when reported, else leave absent without failing.
    - _Requirements: 1.1, 1.3, 1.4, 1.5, 1.6, 1.7, 1.8, 1.9, 1.10_
  - [ ]* 9.2 Write property test for parent attribution (Hypothesis)
    - **Property 20: Parent attribution follows the active span**
    - **Validates: Requirements 1.3, 1.4, 1.10**
  - [ ]* 9.3 Write property test for identifier uniqueness (Hypothesis)
    - **Property 21: Identifier uniqueness**
    - **Validates: Requirements 1.9**
  - [ ]* 9.4 Write property test for error recording and re-raise (Hypothesis)
    - **Property 22: Errors are recorded and re-raised**
    - **Validates: Requirements 1.6**

- [ ] 10. Implement the SDK bounded queue and background sender
  - [ ] 10.1 Implement `BoundedQueue` (drop-oldest) (manual - implemented by the maintainer, not by an agent)
    - Back the queue with a lock-guarded `deque(maxlen=queue_capacity)`; non-blocking enqueue that evicts the oldest item when full.
    - _Requirements: 2.1, 2.4_
  - [ ] 10.2 Implement `BackgroundSender` with batching, retry, and suppression (manual - implemented by the maintainer, not by an agent)
    - Daemon thread that flushes when queued spans reach `batch_size` or `flush_interval` elapses; group spans by `trace_id`; POST to the ingestion endpoint with the API key; retry up to `max_retries` then discard; retain data when backend unreachable; suppress all transmit-path exceptions from the host.
    - _Requirements: 2.2, 2.3, 2.5, 2.6, 2.7_
  - [ ]* 10.3 Write property test for drop-oldest overflow (Hypothesis)
    - **Property 23: Bounded queue drops oldest under overflow**
    - **Validates: Requirements 2.4**
  - [ ]* 10.4 Write property test for bounded retries then discard (Hypothesis, mocked transport)
    - **Property 24: Bounded retries then discard; host unaffected**
    - **Validates: Requirements 2.5, 2.6**
  - [ ]* 10.5 Write property test for transmission carrying the API key (Hypothesis, mocked transport)
    - **Property 25: Transmission carries the API key**
    - **Validates: Requirements 2.7**

- [ ] 11. Implement the SDK public API and shutdown flush
  - [ ] 11.1 Implement `init`, `observe`, `wrap_llm`, `flush`, and atexit hook (manual - implemented by the maintainer, not by an agent)
    - Wire `init` to configure the global client and start the sender; implement `observe` (sync + async) and `wrap_llm` on top of the context/recorder; implement `flush` as a best-effort synchronous drain that never raises; register an `atexit` hook that flushes within `shutdown_timeout` and allows exit regardless of completion.
    - _Requirements: 1.1, 1.2, 1.3, 1.4, 2.1, 2.8_
  - [ ]* 11.2 Write SDK example/unit tests
    - Non-blocking enqueue returns promptly with a stalled sender; batch triggers on size and on interval with a mocked transport; backend-unreachable keeps the host running; atexit flush within timeout.
    - _Requirements: 2.1, 2.2, 2.3, 2.8_

- [ ] 12. Checkpoint - SDK → ingestion → database works end to end
  - [ ] 12.1 Write the end-to-end SDK-to-database integration test
    - Run the real SDK against the running backend and the disposable PostgreSQL fixture (4.4): record a trace via the SDK, flush, then assert the trace and its spans are persisted and retrievable with correct parent references and derived fields.
    - Mandatory: proves the ingestion vertical slice end to end.
    - _Requirements: 1.1, 2.2, 3.6, 3.8_
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 13. Implement the query API: cursor, listing, filtering, and detail
  - [ ] 13.1 Implement keyset cursor encode/decode
    - Implement opaque base64url cursor over `{ start_time, trace_id }`; malformed cursor → 400 `invalid_cursor`.
    - _Requirements: 5.1, 5.11_
  - [ ] 13.2 Implement `GET /v1/traces` listing with filters and pagination
    - Project-scoped list ordered by `start_time DESC`, `trace_id DESC`, page size default 50 / max 100, with `next_cursor` when more rows remain; include derived status, cost, latency, start/end time, span count per row.
    - Apply `status` filter (invalid value → 400 `invalid_filter`), inclusive `from`/`to` time-range filter (invalid/`from > to` → 400), and combined filters conjunctively.
    - _Requirements: 5.1, 5.3, 5.4, 5.5, 5.6, 5.7, 5.11, 5.12_
  - [ ] 13.3 Implement `GET /v1/traces/{trace_id}` detail
    - Return the trace with its spans as a flat list including every field in Req 5.8 plus per-span `latency_ms`, `cost`, and `price_unknown`; unknown or cross-project trace → 404 (indistinguishable).
    - _Requirements: 5.8, 5.9, 5.10, 6.7_
  - [ ]* 13.4 Write property test for ordered, partitioning pagination (Hypothesis, test DB)
    - **Property 16: Listing is ordered and pages partition the result set**
    - **Validates: Requirements 5.1, 5.11**
  - [ ]* 13.5 Write property test for status filter correctness (Hypothesis, test DB)
    - **Property 17: Status filter correctness**
    - **Validates: Requirements 5.4**
  - [ ]* 13.6 Write property test for time-range filter correctness (Hypothesis, test DB)
    - **Property 18: Time-range filter correctness**
    - **Validates: Requirements 5.6**
  - [ ]* 13.7 Write property test for list/detail field completeness (Hypothesis, test DB)
    - **Property 19: Trace list and detail field completeness**
    - **Validates: Requirements 5.3, 5.8**
  - [ ]* 13.8 Write property test for project isolation (Hypothesis, test DB)
    - **Property 13: Project isolation**
    - **Validates: Requirements 6.6**
  - [ ]* 13.9 Write property test for cross-project/unknown fetch not-found (Hypothesis, test DB)
    - **Property 15: Cross-project or unknown trace fetch yields not-found**
    - **Validates: Requirements 5.9, 5.10, 6.7**

- [ ] 14. Implement the Admin CLI
  - [ ] 14.1 Implement `backend/admin.py` commands
    - Implement `project create --id` (unique id, generate raw key via `secrets.token_urlsafe`, store only SHA-256 hash, print raw key once; duplicate id → error with no duplicate), `project list` (no secrets), and `project rotate-key --id` (new key printed once, old hash invalidated). Never log the raw key.
    - _Requirements: 7.1, 7.2, 7.3, 7.4_
  - [ ]* 14.2 Write unit tests for the Admin CLI
    - Create prints the raw key once and stores only the hash; duplicate-id create fails with no duplicate; rotate-key issues a new key and invalidates the old hash.
    - _Requirements: 7.1, 7.2, 7.3, 7.4_

- [ ] 15. Checkpoint - backend API and provisioning
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 16. Implement the dashboard span-tree reconstruction
  - [ ] 16.1 Implement the pure `buildSpanTree(spans)` function
    - Implement the precise reconstruction algorithm: index by `span_id`; treat absent/self-referencing/dangling parents as roots; break cycles by promoting the first span whose parent is already an ancestor; order children (and roots) by ascending `start_time`, ties by ascending `span_id`; include every span exactly once.
    - _Requirements: 10.1, 10.2, 10.3, 10.4, 10.5, 10.6, 10.7, 9.1_
  - [ ]* 16.2 Write property test for forest well-formedness (fast-check)
    - **Property 1: Forest well-formedness**
    - **Validates: Requirements 10.2, 10.3, 10.4, 10.5, 10.6**
  - [ ]* 16.3 Write property test for non-root parent placement (fast-check)
    - **Property 2: Non-root parent placement**
    - **Validates: Requirements 10.1, 9.1**
  - [ ]* 16.4 Write property test for deterministic sibling ordering (fast-check)
    - **Property 3: Deterministic sibling ordering**
    - **Validates: Requirements 10.7**
  - [ ] 16.5 Write example/unit tests for span-tree reconstruction
    - Single-span trace, multi-level nested trace, and a span referencing a missing parent reconstruct to the expected structure and ordering.
    - Mandatory: part of the Requirement 12 verification gate.
    - _Requirements: 12.5_

- [ ] 17. Implement the dashboard API client and key-entry gate
  - [ ] 17.1 Implement the typed Query API client and API-key gate
    - Implement a typed client for `GET /v1/traces` and `GET /v1/traces/{id}` sending the key in the `X-API-Key` header; implement the API-key entry screen that blocks any trace call until a key is entered and scopes all reads to the resolved project.
    - _Requirements: 8.2, 8.3_
  - [ ]* 17.2 Write component tests for the API-key gate
    - No trace call is made before a key is entered; entering a key scopes subsequent requests.
    - _Requirements: 8.2, 8.3_

- [ ] 18. Implement the dashboard trace list
  - [ ] 18.1 Implement the Trace List view with filters and pagination
    - Render rows with trace id, start time, status, cost, and latency ordered most-recent-first; wire status and time-range filters (individually and combined) to query params; render the empty state with no rows when nothing matches; show a load error that retains the entered key and filter selections; implement load-more via the continuation token; open the Trace Detail view on row selection.
    - _Requirements: 8.1, 8.4, 8.5, 8.6, 8.7, 8.8, 8.9, 8.10_
  - [ ]* 18.2 Write component tests for the Trace List
    - Rows and ordering, empty-state, load-error retaining key+filters, filter-to-query-param wiring, load-more via cursor, and row selection opening detail.
    - _Requirements: 8.1, 8.4, 8.5, 8.6, 8.7, 8.8, 8.9, 8.10_

- [ ] 19. Implement the dashboard trace detail and span-tree view
  - [ ] 19.1 Implement the Trace Detail view
    - Reconstruct the span tree via `buildSpanTree` and render children beneath parents; show per-span name, type, status, latency, and cost; show trace cost and latency; hide the span detail panel while no span is selected; on selection show input, output, model, prompt/completion token counts, error, and metadata, indicating absent fields; display the span error for error spans; display "unknown price" when `price_unknown` is set; show a load error without rendering a tree when the trace fails to load.
    - _Requirements: 9.1, 9.2, 9.3, 9.4, 9.5, 9.6, 9.7, 9.8, 9.9_
  - [ ]* 19.2 Write component tests for the Trace Detail view
    - Span-tree nesting, per-span fields, trace cost/latency, no-selection hides the panel, selected-span details with absent-field indicators, error display, "unknown price" rendering, and load-error without a tree.
    - _Requirements: 9.2, 9.3, 9.4, 9.5, 9.6, 9.7, 9.8, 9.9_

- [ ] 20. Checkpoint - dashboard complete
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 21. Wire local deployment with Docker Compose
  - [ ] 21.1 Author Dockerfiles and `docker-compose.yml`
    - Add a backend Dockerfile, a dashboard Dockerfile, and a Compose file defining Backend, Database (PostgreSQL), and Dashboard services with health checks and readiness reporting within 120 seconds; configure the backend to create schema on first startup against an empty database before serving; wire the SDK endpoint and dashboard-to-query-API connectivity.
    - Record an ADR under `docs/adr/` for the Compose topology/readiness decision.
    - _Requirements: 11.1, 11.2, 11.3, 11.4, 11.6, 12.8_
  - [ ]* 21.2 Write smoke and integration tests for the Compose setup
    - Smoke: services start, schema is created on first startup against an empty database, `/healthz` reports ready. Integration: the SDK reaches the ingestion endpoint and the dashboard reaches the query API while ready; a service failing to become ready within the timeout is reported by name with a reason.
    - _Requirements: 11.2, 11.3, 11.5, 11.6_

- [ ] 22. Set up GitHub Actions CI
  - [ ] 22.1 Author the CI workflow YAML
    - Add a GitHub Actions workflow that runs on every push and pull request, spinning up a PostgreSQL service for DB-touching tests; run `pytest` + Hypothesis for the backend and SDK, and `vitest` + fast-check for the dashboard; fail the build on any test failure.
    - Supports the Requirement 12 verification gate (12.7); no requirement maps directly.
    - _Requirements: 12.7_
  - [ ]* 22.2 Add a CI status badge and caching
    - Add dependency caching for Python and Node, and a workflow-status badge to the README.
    - _Requirements: 12.7_

- [ ] 23. Build demo application and README
  - [ ] 23.1 Implement the SDK demo script and project README
    - Write a small script using the SDK that produces realistic traces, including nested agent → tool → llm spans and an error case, against the running backend.
    - Write a README with an architecture diagram, a quickstart, and screenshots.
    - Illustrative only; exercises the SDK span-recording and transport surface.
    - _Requirements: 1.1, 1.2, 1.3, 1.4, 1.5, 1.6, 1.7, 1.8, 1.9, 1.10, 2.1, 2.2, 2.8_

- [ ] 24. Final checkpoint - full verification gate
  - Ensure all tests pass, ask the user if questions arise. A build is complete only when the mandatory Requirement 12 tests pass: cost in-table and missing-model (2.4), ingestion valid/invalid and idempotency (7.5), span-tree reconstruction cases (16.5), and the SDK-to-database end-to-end integration test (12.1).
  - _Requirements: 12.7_

## Notes

- Tasks marked with `*` are optional test sub-tasks and can be skipped for a faster MVP; core implementation sub-tasks are never optional.
- The following test sub-tasks are **mandatory** (no `*`) because they form the Requirement 12 verification gate: 2.4 (cost in-table / missing-model), 7.5 (ingestion valid/invalid/idempotent), 7.6 (multi-batch trace ingestion), 16.5 (span-tree reconstruction cases), and 12.1 (SDK-to-database end-to-end). The final gate (Task 24) depends on 2.4, 7.5, 16.5, and 12.1.
- The 25 correctness properties are each implemented by a single property-based test: Hypothesis for backend/SDK, fast-check for the dashboard; these property tests remain optional (`*`).
- Every database-touching test depends on the disposable PostgreSQL fixture built in 4.4 (transactional rollback or fresh schema per test): 7.2, 7.3, 7.4, 7.5, 7.6, 12.1, and 13.4–13.9. The fixture task 4.4 is mandatory and precedes all of them.
- ADRs are seeded in Task 1 as a template plus seven **"Status: Draft" stub files**; each stub is rewritten into a final ADR (Context / Decision / Consequences / Alternatives considered, in the maintainer's own words) in the same change set that implements the decision it describes (Req 12.8).
- The plan proves a vertical slice first: scaffold → cost/derivation → models/schema/fixture → auth → ingest validation → ingestion endpoint → SDK → SDK-to-database end-to-end checkpoint, before the query API, Admin CLI, dashboard, and Compose; CI and the demo come near the end and the full verification gate is last.
- The following implementation sub-tasks are **manual — implemented by the maintainer, not by an agent** — but remain tracked here for dependencies: 2.1 (cost service), 3.1 (derivation), 4.1 (models), 9.1 (SDK context/recorder), 10.1 (bounded queue), 10.2 (background sender), and 11.1 (SDK public API).
- Checkpoints provide incremental validation at component boundaries.

## Task Dependency Graph

```json
{
  "waves": [
    { "id": 0, "tasks": ["1.1"] },
    { "id": 1, "tasks": ["2.1", "3.1", "4.1", "9.1", "10.1", "16.1", "17.1"] },
    { "id": 2, "tasks": ["2.2", "2.3", "2.4", "3.2", "4.2", "4.4", "5.1", "6.1", "9.2", "9.3", "9.4", "10.2", "16.2", "16.3", "16.4", "16.5", "17.2"] },
    { "id": 3, "tasks": ["4.3", "5.2", "5.3", "6.2", "6.3", "7.1", "10.3", "10.4", "10.5", "11.1", "18.1", "19.1"] },
    { "id": 4, "tasks": ["7.2", "7.3", "7.4", "7.5", "7.6", "11.2", "18.2", "19.2"] },
    { "id": 5, "tasks": ["12.1"] },
    { "id": 6, "tasks": ["13.1", "13.3", "14.1"] },
    { "id": 7, "tasks": ["13.2", "13.4", "13.5", "13.6", "13.7", "13.8", "13.9", "14.2", "21.1", "23.1"] },
    { "id": 8, "tasks": ["21.2", "22.1"] },
    { "id": 9, "tasks": ["22.2"] }
  ]
}
```
