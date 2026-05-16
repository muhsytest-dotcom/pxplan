# Variants Stabilization Progress

Last updated: 2026-05-16

## Completed Slice 1: Core Domain Foundation

Implemented a shared variants domain foundation across backend and frontend.

Backend:
- Added `app/modules/catalog/variant_domain.py`.
- Added canonical variant identity helpers:
  - trim/case-normalize option value IDs
  - sort IDs deterministically
  - reject empty and duplicate value sets
  - generate deterministic SHA-256 identity hashes
- Added structure lifecycle values: `idle`, `dirty`, `synced`.
- Added variant lifecycle contract values: `active`, `archived`, `deleted`.
- Added centralized active/terminal job status guards.
- Added `timeout` to backend job status and phase enums.
- Wired variant generation to use the shared canonical key builder.
- Replaced duplicated active-job status checks with shared guard constants.

Frontend:
- Added `lib/catalog/variant-domain.ts`.
- Added matching canonical variant identity helpers.
- Added matching lifecycle/status types, including `timeout`.
- Updated `variant-matrix.ts` to use the shared canonical key logic.
- Updated variant action blocking to respect active queued/running jobs consistently.

Tests:
- Added backend domain tests in `tests/test_catalog_variant_domain.py`.
- Added frontend domain tests in `lib/catalog/__tests__/variant-domain.test.ts`.

## Completed Slice 2: SSE Job Event Infrastructure

Implemented the first production-safe SSE infrastructure layer while keeping the existing polling flow as fallback.

Backend:
- Added `app/modules/catalog/variant_events.py`.
- Added `VariantJobEventBus` with:
  - monotonic sequence numbers per job
  - replayable in-memory event history
  - typed event names: `state`, `completed`, `failed`, `cancelled`
  - SSE formatting with `id`, `event`, and JSON `data`
  - heartbeat support
  - idle timeout support
- Published job events from the existing job lifecycle update path.
- Added authenticated admin SSE endpoint:
  - `GET /catalog/admin/variant-jobs/{job_id}/events`
  - supports `last_event_id` replay cursor
  - validates store ownership before streaming
  - returns `text/event-stream`
  - sends `Cache-Control: no-cache` and `X-Accel-Buffering: no`

Frontend:
- Added `lib/catalog/variant-job-events.ts`.
- Added event stream URL builder.
- Added typed event payload parser.
- Added `EventSource` opener with credentials enabled.
- Added typed event payloads to `lib/catalog/types.ts`.

Tests:
- Added backend event bus tests in `tests/test_catalog_variant_events.py`.
- Added frontend event helper tests in `lib/catalog/__tests__/variant-job-events.test.ts`.

## Completed Slice 3: Variant Explosion Protection & Performance Foundation

Implemented industrial-grade protection against variant explosion and established the foundation for high-performance async processing.

Backend:
- **Phase 3 — Variant Explosion Protection**:
  - Defined strict structural limits: 10 options max, 100 values per option.
  - Implemented **Tier-Aware Quotas**: Basic stores (5K variants max), Pro stores (50K variants max).
  - Added `check_variant_explosion_risk` guard to prevent combinatorial explosion before it reaches the DB.
  - Enforced quotas in `create_product_option`, `create_product_option_value`, and `preview_rebuild_variants_impact`.
  - Added readiness check in `create_catalog_variant_job_for_product` to block invalid job submissions.
- **Phase 4 — Performance & Resilience**:
  - Implemented **Job Cancellation API**: Users can now cancel `QUEUED` or `RUNNING` jobs. The worker checks for cancellation at each step of the generation loop to abort promptly.
  - Implemented **Job Timeout Cleanup**: `cleanup_stale_variant_jobs` marks stuck jobs as `TIMEOUT` after 30 minutes of inactivity.
  - Optimized **Paginated Variant Listing**: Ensured frontend/backend consistently use paginated fetching (50 per page default) for large variant sets.
  - Added `tier` field to `Store` model to support subscription-based scaling.

## Completed Slice 4: Architectural Consolidation & Durable Events

Finalized the core architectural stabilization by refactoring the frontend state and ensuring durable backend observability.

Backend:
- **Phase 6 — Durable Event History**:
  - Added `CatalogVariantJobEvent` model to persist all job state transitions.
  - Refactored `VariantJobEventBus` to support sequence synchronization and DB-backed replay.
  - Updated SSE endpoints to fetch historical events from DB if they fall out of memory (deque maxlen).
  - Integrated persistent event creation into `_mark_catalog_variant_job` (via `_transition_variant_job`).

Frontend:
- **Phase 5 — Frontend State Architecture**:
  - Transitioned `VariantStructureStudio` to a React Context-based state architecture.
  - Eliminated complex prop-drilling chain for 40+ props.
  - Refactored monolithic component into modular functional sub-components: `StudioHeader`, `StudioOptionsList`, `StudioValueManager`, `StudioImpactPreview`, `StudioFooter`, and `StudioRebuildModal`.
  - Centralized derived selectors (e.g., `isBusy`, `effectiveOperationalState`) into the context provider for performance.

## Completed Slice 5: Worker Process Boundary (Phase 7)

Finalized the decoupled worker architecture to ensure long-running variant tasks do not block the API lifecycle.

Backend:
- **Decoupled Worker Process**: Implemented `PX-B/app/modules/catalog/worker.py` as a standalone background worker.
- **Atomic Job Claiming**: Implemented robust job locking using transactional status updates (`QUEUED` -> `RUNNING`).
- **API Delegation**: Updated `catalog/router.py` to only enqueue jobs, delegating all execution responsibility to the worker.
- **Infrastructure**: Added `make run-worker` and `make dev-worker` targets to the `Makefile`.
- **Durable Streaming**: Verified that the worker correctly emits events via the durable SSE event history for real-time frontend feedback.

## Completed Slice 6: Atomic Snapshot Tables (Phase 8)

Implemented the service-layer and storage architecture for immutable product structure snapshots and atomic publication swaps.

Backend:
- **Atomic Snapshot Capture**: Implemented `CatalogProductStructureSnapshot` model and service logic to capture full structural state (options, values, translations) at job start.
- **Multi-Version Concurrency**: Updated `ProductVariant` with a `version` field and adjusted unique constraints to allow variants from different snapshots to coexist.
- **Worker Isolation**: Updated the variant generation worker to operate against immutable snapshots instead of live tables, ensuring structural integrity during concurrent edits.
- **Atomic Swap API**: Implemented `publish_catalog_product_snapshot` to trigger zero-downtime storefront structural updates by flipping the `published_version` pointer.
- **Storefront Authoritative Structure**: Added a storefront structure resolution service that prioritizes published snapshots for consistent UI rendering.
- **Administrative API**: Added endpoints to list snapshots and manage publication status.

## Completed Slice 7: i18n Framework, Error Classification, and Worker Retry

Implemented the three BLOCKER items from the implementation guides while preserving the existing worker, snapshot, SSE, and custom frontend i18n architecture.

Backend:
- Added `app/core/i18n_keys.py` as the central catalog variant/product i18n key registry.
- Extended `AppException` and HTTP exception handling so API error envelopes can carry `error.i18n_key` and interpolation `context`.
- Added `app/modules/catalog/job_errors.py` with `JobErrorType`, retryable/non-retryable classification sets, concrete job error classes, and `wrap_error()`.
- Added `error_type`, `retry_count`, `last_error_at`, and `last_error_message` to `CatalogVariantJob`, plus Alembic migration `b8f7a2c9d401_add_variant_job_error_classification.py`.
- Updated job execution to classify validation, stale-structure, cancellation, timeout, database, and unknown failures; failed job events now expose the stored `error_type`.
- Implemented worker retry with bounded backoff for retryable `JobError`s and immediate terminal handling for validation/conflict/cancelled failures.
- Fixed snapshot job creation by capturing a durable structure snapshot before queuing a job.
- Restored compatibility routes and test hooks for existing `generate-missing` job endpoints and event bus publishing.

Frontend:
- Added `lib/catalog/i18n.ts` with catalog error-key translation and interpolation support that works with the existing locale/path i18n system.
- Updated API error adapters and shared auth error typing to preserve backend i18n metadata.
- Added frontend tests for catalog i18n-key translation and fallback behavior.
- Kept `VariantStructureStudio` compatible with both context-provider usage and the established props-based tests/call sites.
- Improved active job recovery polling so fallback polling checks the current job immediately before starting the interval.

Tests and validation:
- Added backend tests in `tests/test_job_error_classification.py` for retryability, concrete job errors, wrapping, and worker retry behavior.
- Updated existing backend variant job tests for the decoupled worker boundary and readiness checks.
- Updated frontend API and variant studio tests to match current contracts.
- Verified full backend tests, full frontend tests, backend ruff, backend mypy, frontend lint, frontend typecheck, frontend i18n check, and frontend production build.

## Verification Completed

Backend:
- Full backend tests: passed
- Variant/snapshot regression suite: `43 passed`
- Domain protection/quota tests: `15 passed`
- Job operation tests (cancellation/timeout): Added `tests/test_catalog_variant_ops.py`
- Durable events verification: Verified `CatalogVariantJobEvent` creation and SSE replay logic.
- Worker atomicity: Verified atomic claiming logic for concurrent worker support.
- Snapshot Integrity: Verified snapshot capture and reconstruction logic.
- Full backend ruff/mypy: passed

Frontend:
- Full frontend tests: `293 passed`
- Context architecture verified: Successful refactor of `VariantStructureStudio` without regressions.
- Pagination and SSE integration verified.
- Frontend typecheck, lint, i18n check, and production build passed.

## Still Not Complete

The full multi-phase plan is not finished yet. Remaining major areas:
- **Phase 9: Observability & Metrics**: Adding an observability dashboard for job performance metrics and health.
- **Phase 10: Policy-Based Tier Enforcement**: Broader enforcement of store tiers across all catalog operations.
- **Archive-vs-delete migration**: Finalizing the strategy for existing variants and historical references.
- **E2E coverage**: Broader E2E coverage for critical admin and storefront workflows.
- **Dead-code cleanup**: Final pass across both apps after all phases are complete.
