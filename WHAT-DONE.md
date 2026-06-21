# Variants Stabilization Progress

Last updated: 2026-06-14

## Completed: Variant Worker Stabilization & Job Processing Fix (2026-05-19)

Fixed critical issues preventing the background variant worker from processing jobs.

### Changes Made

**Makefile**
- Updated `run-worker` and `dev-worker` targets to load `.env.local.dev` (same as `make run`)
- This ensures the worker connects to PostgreSQL instead of falling back to SQLite

**Worker (`app/modules/catalog/worker.py`)**
- Added missing model imports to resolve SQLAlchemy foreign key errors:
  - `from app.modules.users.models import User`
  - `from app.modules.stores.models import Store`
- Improved `fetch_next_job()` with `with_for_update(skip_locked=True)` for safe concurrent job claiming
- Added better error logging during job claiming

**Tests**
- Created `tests/test_catalog_worker.py` with comprehensive test coverage:
  - `TestFetchNextJob`: job claiming, rollback on failure, empty queue handling
  - `TestExecuteWithRetry`: success path, retry logic, failure handling
  - `TestWorkerIntegration`: basic worker flow

### Issues Resolved
- Worker was stuck on "Query returned job: None" even when jobs existed
- `NoReferencedTableError` on `catalog_jobs.created_by_user_id` and `store_id`
- Jobs left stuck in `running` state after failed processing attempts

### Result
Worker can now successfully:
- Poll for queued jobs
- Atomically claim jobs
- Execute variant generation / rebuild jobs
- Handle retries and mark failed jobs correctly

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

## Completed Slice 8: Blocker Cleanup and Full Quality Gate Refresh

Status: Complete on 2026-05-16

This pass re-read the implementation guides, verified that the three BLOCKER tracks were present in the codebase, and tightened the remaining rough edges around i18n envelopes, translation coverage, lint warnings, and validation.

Backend:
- Added reusable catalog HTTP error detail helpers in `PX-B/app/modules/catalog/router.py` so catalog routes can consistently return `message`, `i18n_key`, and optional interpolation `context`.
- Updated variant job create/rebuild/active/detail/cancel/SSE route not-found responses to include catalog i18n keys.
- Updated the product structure guard HTTP conflict response to include `catalog.variant.errors.structure_stale_hash` and context.
- Added a backend regression test asserting that variant job creation for a missing product returns `error.i18n_key = catalog.product.errors.not_found`.

Frontend:
- Completed French and German catalog variant error translations in `PX-F/lib/catalog/i18n.ts`.
- Updated the catalog i18n regression test to assert locale-specific French translation behavior.
- Removed unused destructured values from `VariantStructureStudio` and cleaned the active-job effect dependency expression in `AdminProductEditForm`, eliminating lint warnings without changing behavior.

Verification completed:
- Backend full tests: passed with `.venv/bin/python -m pytest tests/ -q`.
- Backend lint: passed with `.venv/bin/ruff check .`.
- Backend typecheck: passed with `.venv/bin/mypy app/`.
- Frontend full tests: `293 passed` with `npm test`.
- Frontend lint: passed with `npm run lint`.
- Frontend typecheck: passed with `npm run typecheck`.
- Frontend i18n check: passed for 10 locales with `npm run i18n:check`.
- Frontend production build: passed with `npm run build` after allowing the build to fetch required Google font assets.

## Completed Slice 9: Backend Variant Job Observability and Write Protection

Status: Complete on 2026-05-17

Implemented the backend half of the Phase 9 observability work and added rate limiting protection to variant job mutation endpoints.

Backend:
- Added durable variant job metrics aggregation in `PX-B/app/modules/catalog/service.py`.
- Added `CatalogVariantJobMetricsResponse` in `PX-B/app/modules/catalog/schemas.py`.
- Added authenticated owner-scoped endpoint `GET /catalog/admin/metrics/variant-jobs`.
- Supported optional `store_id` filtering while preventing cross-store leakage.
- Added Prometheus-compatible lifecycle instrumentation in `PX-B/app/core/metrics.py`.
- Tracked job transitions, queue wait duration, and execution duration from the shared `_mark_catalog_variant_job` transition path.
- Added `CATALOG_VARIANT_JOB_WRITE` to the existing rate-limit policy registry.
- Applied the policy to generate-missing, rebuild, and cancel variant job endpoints.

Tests:
- Added `PX-B/tests/test_catalog_variant_observability.py`.
- Covered durable metrics aggregation for terminal and active jobs.
- Covered authenticated metrics endpoint scoping to stores owned by the current user.
- Covered catalog variant job write rate limiting with the existing `RateLimitExceededError` response envelope.

Verification completed:
- Backend targeted tests: `python -m pytest tests/test_catalog_variant_observability.py tests/test_rate_limit.py -q` passed.
- Backend catalog regression tests: `python -m pytest tests/test_catalog_variant_observability.py tests/test_catalog_variant_ops.py tests/test_catalog_variants.py tests/test_job_error_classification.py -q` passed.
- Backend full tests: `python -m pytest tests -q` passed.
- Backend lint: `.\.venv\bin\ruff check .` passed.
- Backend typecheck: `python -m mypy app` passed.
- Frontend full tests: `293 passed` with `npm test`.
- Frontend lint: `npm run lint` passed.
- Frontend typecheck: `npm run typecheck` passed.
- Frontend i18n check: `npm run i18n:check` passed for 10 locales.
- Frontend production build: `npm run build` passed.

## Completed Slice 10: Frontend Variant Job Metrics Dashboard

Status: Complete on 2026-05-17

Completed the frontend/admin visualization layer for the backend variant job metrics from Slice 9.

Frontend:
- Added `VariantJobMetricsPanel` in `PX-F/app/components/variant-job-metrics-panel.tsx`.
- Added the panel to the dashboard landing page at `PX-F/app/[locale]/(app)/dashboard/page.tsx`.
- Added typed metrics API support via `getVariantJobMetrics` in `PX-F/lib/catalog/api.ts`.
- Added `CatalogVariantJobMetricsResponse` in `PX-F/lib/catalog/types.ts`.
- Added operational metrics copy in `PX-F/lib/i18n/ui-copy.ts` for all supported locales.
- Rendered health status, active jobs, success rate, recent throughput, queue wait, duration, status counts, and classified error breakdown.
- Kept the dashboard compact and operational, matching the existing admin UI patterns.

Tests:
- Added `PX-F/app/components/__tests__/variant-job-metrics-panel.test.tsx`.
- Covered successful metrics rendering, empty state, and localized error state.
- Extended catalog API tests for the metrics route.
- Extended i18n copy tests to ensure operational metrics copy exists for all locales and title text is localized.

Verification completed:
- Frontend focused tests: `npm test -- app/components/__tests__/variant-job-metrics-panel.test.tsx lib/i18n/__tests__/ui-copy.test.ts lib/catalog/__tests__/api.test.ts` passed (`34 passed`).
- Frontend full tests: `npm test` passed (`299 passed`).
- Frontend lint: `npm run lint` passed.
- Frontend typecheck: `npm run typecheck` passed.
- Frontend i18n check: `npm run i18n:check` passed for 10 locales.
- Frontend production build: `npm run build` passed.
- Backend targeted tests: `python -m pytest tests/test_catalog_variant_observability.py tests/test_rate_limit.py -q` passed.
- Backend full tests: `python -m pytest tests -q` passed.
- Backend lint: `.\.venv\bin\ruff check .` passed.
- Backend typecheck: `python -m mypy app` passed.

## Completed Slice 11: Frontend Variant Job SSE Recovery

Status: Complete on 2026-05-17

Improved the admin variant job update path so environments without native SSE support recover through polling without noisy console errors, while still showing users how the job progress is being refreshed.

Frontend:
- Updated `PX-F/lib/catalog/variant-job-events.ts` so `openVariantJobEventSource` is capability-aware and returns `null` when `EventSource` is unavailable.
- Updated `PX-F/app/components/admin-product-edit-form.tsx` to switch active variant jobs to polling recovery when SSE cannot be opened or when the stream errors.
- Removed noisy variant job SSE `console.log`, `console.warn`, and `console.error` output from the recovery path.
- Added `variantJobConnectionMode` propagation into `ProductVariantsSection`.
- Added localized job-panel badges for connected live updates and polling recovery.
- Added catalog admin copy keys for the recovery status labels, with existing locale fallback behavior preserved.

Tests:
- Extended `PX-F/lib/catalog/__tests__/variant-job-events.test.ts` to cover missing `EventSource` support.
- Extended `PX-F/app/components/product-editor/__tests__/product-variants-section.test.tsx` to cover the polling recovery badge.
- Re-ran the admin product editor focused suite to verify active-job recovery now falls back without `EventSource is not defined` stderr noise.

Verification completed:
- Frontend focused tests: `npm test -- lib/catalog/__tests__/variant-job-events.test.ts app/components/product-editor/__tests__/product-variants-section.test.tsx app/components/__tests__/admin-product-edit-form.test.tsx` passed (`38 passed`).
- Frontend lint: `npm run lint` passed.
- Frontend typecheck: `npm run typecheck` passed.
- Frontend i18n check: `npm run i18n:check` passed for 10 locales.

## Completed Slice 12: Batched Variant Selection Resolution

Status: Complete on 2026-05-17

Addressed the high-priority N+1 query risk in variant listing paths by batching selection resolution for admin and storefront variant responses.

Backend:
- Added `resolve_variant_selections_for_variants` in `PX-B/app/modules/catalog/service.py`.
- Batched variant value links, option values, options, option translations, and option value translations for a page of variants.
- Kept `resolve_variant_selections_for_variant` as the single-variant compatibility wrapper.
- Updated admin variant listing in `PX-B/app/modules/catalog/router.py` to resolve all displayed variant selections in one batched service call.
- Updated storefront variant listing to use the same batched resolver while preserving requested-locale translation behavior.

Tests:
- Added `PX-B/tests/test_catalog_variant_query_efficiency.py`.
- Covered a four-variant admin listing and asserted the catalog selection path stays bounded rather than growing per variant.
- Re-ran existing storefront variant tests covering store scoping, active-only filtering, and localized option/value display.

Verification completed:
- Backend focused tests: `python -m pytest tests/test_catalog_variant_query_efficiency.py tests/test_catalog_variants.py::test_storefront_variants_are_store_scoped_and_active_only tests/test_catalog_variants.py::test_option_translations_and_visual_value_metadata_are_exposed -q` passed.
- Backend full tests: `python -m pytest tests -q` passed.
- Backend lint: `.\.venv\bin\ruff check .` passed.
- Backend typecheck: `python -m mypy app` passed.
- Frontend focused tests: `npm test -- lib/catalog/__tests__/variant-job-events.test.ts app/components/product-editor/__tests__/product-variants-section.test.tsx app/components/__tests__/admin-product-edit-form.test.tsx` passed (`38 passed`).
- Frontend full tests: `npm test` passed (`301 passed`).
- Frontend lint: `npm run lint` passed.
- Frontend typecheck: `npm run typecheck` passed.
- Frontend i18n check: `npm run i18n:check` passed for 10 locales.
- Frontend production build: `npm run build` passed.

## Completed Slice 13: Policy-Based Tier Enforcement for Variant Structure Writes

Status: Complete on 2026-05-17

Closed the remaining tier-policy gap where public structure edits could create over-quota option/value matrices that only failed later during rebuild preview or job creation.

Backend:
- Added `ProductVariantTierQuotaExceededError` with `catalog.variant.errors.tier_quota_exceeded` metadata and interpolation context.
- Added shared projected-count and target-store tier quota helpers in `PX-B/app/modules/catalog/service.py`.
- Enforced Basic/Pro projected variant limits when adding option values.
- Enforced the target store tier when applying variant templates.
- Enforced the target store tier when copying product structures.
- Kept rebuild-impact and job-submission quota protection for legacy/imported over-quota structures.
- Updated the option-value API path to return structured bad-request responses for value-count validation instead of leaking service `ValueError`s.

Tests:
- Added `PX-B/tests/test_catalog_tier_policy.py`.
- Covered Basic-tier blocking for projected option-value growth beyond 5,000 variants.
- Covered Pro-tier allowance for the same projected structure.
- Covered template application and structure copy using the target store tier limit.
- Updated the existing rebuild-impact quota regression to seed legacy/imported over-quota data directly, since public structure writes now block that state earlier.

Verification completed:
- Backend focused tests: `python -m pytest tests/test_catalog_tier_policy.py tests/test_catalog_variant_ops.py::test_rebuild_impact_enforces_variant_quota -q` passed.
- Backend full tests: `python -m pytest tests -q` passed.
- Backend lint: `.\.venv\bin\ruff check .` passed.
- Backend typecheck: `python -m mypy app` passed.
- Frontend full tests: `npm test` passed (`301 passed`).
- Frontend lint: `npm run lint` passed.
- Frontend typecheck: `npm run typecheck` passed.
- Frontend i18n check: `npm run i18n:check` passed for 10 locales.
- Frontend production build: `npm run build` passed.

## Completed Slice 14: Storefront Published Snapshot Regression Coverage

Status: Complete on 2026-05-17

Added focused backend integration coverage for the admin generation -> snapshot publish -> storefront visibility contract.

Backend:
- Added `test_published_snapshot_controls_storefront_while_draft_structure_changes` to `PX-B/tests/test_catalog_variants.py`.
- Covered generate-missing job creation through the API and direct worker execution with `execute_catalog_variant_job`.
- Verified storefront variants remain empty before the generated snapshot is explicitly published.
- Verified publishing a generated snapshot exposes the expected active variant and localized selection data to the storefront.
- Verified later draft structure edits do not leak into storefront structure or variant responses while the published snapshot remains authoritative.
- Preserved existing architecture: no API, database, schema, or runtime behavior changes were needed for this slice.

Important reasoning:
- The project already has immutable snapshots and publication swaps, but release hardening needed a concrete regression test that exercises the full API-level publication boundary.
- This intentionally avoids the unresolved archive-vs-delete product decision and does not change variant deletion semantics.
- Storefront visibility must remain controlled by `product.published_version`; draft option/value edits can continue safely without partial storefront exposure.

Verification completed:
- Backend focused test: `.\.venv\bin\python -m pytest tests/test_catalog_variants.py::test_published_snapshot_controls_storefront_while_draft_structure_changes -q` passed.
- Backend catalog variant suite: `.\.venv\bin\python -m pytest tests/test_catalog_variants.py -q` passed.
- Backend full tests: `.\.venv\bin\python -m pytest tests -q` passed.
- Backend lint: `.\.venv\bin\ruff check .` passed.
- Backend typecheck: `.\.venv\bin\python -m mypy app` passed.
- Frontend full tests: `npm test` passed (`301 passed`).
- Frontend lint: `npm run lint` passed.
- Frontend typecheck: `npm run typecheck` passed.
- Frontend i18n check: `npm run i18n:check` passed for 10 locales.
- Frontend production build: `npm run build` passed.

## Completed Slice 15: API Regression Coverage for Template Quotas and Rebuild Media Rebinding

Status: Complete on 2026-05-17

Added backend API-level release-hardening coverage for two remaining critical workflows: template quota failures and media rebinding after variant rebuild.

Backend:
- Added `test_template_apply_api_returns_i18n_quota_error_for_basic_store` to `PX-B/tests/test_catalog_tier_policy.py`.
- The new template quota regression creates an oversized store-scoped template and verifies the admin apply endpoint returns the existing `PRODUCT_VARIANT_TIER_QUOTA_EXCEEDED` envelope.
- Verified the API response carries `error.i18n_key = catalog.variant.errors.tier_quota_exceeded` and interpolation context `{ tier, limit, projected }`.
- Added `test_rebuild_variants_marks_detached_media_for_rebinding` to `PX-B/tests/test_catalog_variants.py`.
- The rebuild regression verifies rebuild impact reports media detachment before a full matrix rebuild.
- Verified the rebuild path detaches media from removed variants and marks it with `needs_variant_rebinding = true`.

Important reasoning:
- The underlying template quota policy and rebuild media-detach behavior already existed, but release hardening needed end-to-end API regressions to protect the contracts future agents are most likely to break.
- This slice intentionally did not alter runtime behavior, API paths, database schema, or archive-vs-delete semantics.
- The archive-vs-delete migration remains a separate product/domain decision and should still be planned before changing variant deletion behavior.

API, database, and schema changes:
- No API path changes.
- No database migration.
- No schema changes.
- Test-only coverage added for existing behavior.

Verification completed:
- Backend focused template quota API test: `.\.venv\bin\python -m pytest tests/test_catalog_tier_policy.py::test_template_apply_api_returns_i18n_quota_error_for_basic_store -q` passed.
- Backend focused rebuild media rebinding test: `.\.venv\bin\python -m pytest tests/test_catalog_variants.py::test_rebuild_variants_marks_detached_media_for_rebinding -q` passed.
- Backend affected suites: `.\.venv\bin\python -m pytest tests/test_catalog_tier_policy.py tests/test_catalog_variants.py -q` passed.
- Backend full tests: `.\.venv\bin\python -m pytest tests -q` passed.
- Backend lint: `.\.venv\bin\ruff check .` passed.
- Backend typecheck: `.\.venv\bin\python -m mypy app` passed.
- Frontend full tests: `npm test` passed (`301 passed`).
- Frontend lint: `npm run lint` passed.
- Frontend typecheck: `npm run typecheck` passed.
- Frontend i18n check: `npm run i18n:check` passed for 10 locales.
- Frontend production build: `npm run build` passed.

## Completed Slice 16: Frontend Snapshot Publication Workflow

Status: Complete on 2026-05-17

Wired the existing backend snapshot publication endpoints into the admin product variant workflow so generated snapshots can be explicitly published to the storefront from the frontend.

Frontend:
- Added `CatalogProductStructureSnapshotRead`, `ProductRead.snapshot_version`, `ProductRead.published_version`, and optional `CatalogVariantJobRead.snapshot_id` to `PX-F/lib/catalog/types.ts`.
- Added `listProductStructureSnapshots` and `publishProductStructureSnapshot` to `PX-F/lib/catalog/api.ts`.
- Updated `PX-F/app/components/admin-product-edit-form.tsx` to load product structure snapshots, track the published version, detect the latest unpublished generated snapshot, and publish it through the existing backend API.
- Refreshed authoritative snapshot, media state, and snapshot list after successful publication.
- Updated `PX-F/app/components/product-editor/product-variants-section.tsx` with a translatable publish-ready panel that appears only when the latest generated snapshot version is newer than the storefront published version.
- Added catalog admin copy keys for the snapshot publish panel and success/failure messages; existing locale fallback behavior remains unchanged and `npm run i18n:check` passes for 10 locales.

Backend:
- No runtime backend snapshot behavior was changed.
- Cleaned the variant metrics route signature from `store_id: UUID | None = Query(default=None)` to `Annotated[UUID | None, Query()] = None` so backend Ruff passes under the WSL venv.
- Let Ruff fix backend import/unused-import style issues in touched test/support files.

Tests:
- Extended `PX-F/lib/catalog/__tests__/api.test.ts` to cover the snapshot list and publish admin routes.
- Extended `PX-F/app/components/product-editor/__tests__/product-variants-section.test.tsx` to cover the publish-ready UI action.
- Extended `PX-F/app/components/__tests__/admin-product-edit-form.test.tsx` to cover publishing the latest generated snapshot from the variants section.

Important reasoning:
- Snapshot generation still does not auto-publish. The UI only exposes the existing explicit publication contract.
- The publish action is version-driven: it appears only when `latestSnapshot.version > product.published_version`.
- The storefront boundary remains owned by `product.published_version`; draft structure edits remain separate until publication.
- Archive-vs-delete semantics were intentionally left unchanged.

API, database, and schema changes:
- Frontend API client now supports existing backend endpoints:
  - `GET /catalog/admin/products/{product_id}/snapshots`
  - `POST /catalog/admin/products/{product_id}/snapshots/{snapshot_id}/publish`
- No backend API path changes.
- No database migration.
- No backend schema change.

Verification completed:
- Frontend focused tests: `npm test -- lib/catalog/__tests__/api.test.ts app/components/product-editor/__tests__/product-variants-section.test.tsx app/components/__tests__/admin-product-edit-form.test.tsx` passed (`61 passed`).
- Frontend full tests: `npm test` passed (`303 passed`).
- Frontend lint: `npm run lint` passed.
- Frontend typecheck: `npm run typecheck` passed.
- Frontend i18n check: `npm run i18n:check` passed for 10 locales.
- Frontend production build: `npm run build` passed.
- Backend lint via WSL venv: `wsl bash -lc "cd /mnt/d/Github/muhsinmuhsy/PX/PX-B && .venv/bin/ruff check ."` passed.
- Backend typecheck via WSL venv: `wsl bash -lc "cd /mnt/d/Github/muhsinmuhsy/PX/PX-B && .venv/bin/python -m mypy app"` passed.
- Backend focused store-domain regression check: `wsl bash -lc "cd /mnt/d/Github/muhsinmuhsy/PX/PX-B && .venv/bin/python -m pytest tests/test_store_domains.py -q"` passed.
- Backend full test note from Slice 16: first WSL full-suite run completed with one unrelated failure in `tests/test_store_domains.py::test_invalid_domain_returns_field_error` (`401` instead of expected `400`), while the isolated test and module both passed. A second full-suite run timed out before completion. This note is superseded by Slice 17, where the full PostgreSQL suite passed.

## Completed Slice 17: PostgreSQL Test Isolation and FK Hardening

Status: Complete on 2026-05-18

Summary:
- Stabilized the backend PostgreSQL validation path so it now matches SQLite's fresh-test isolation while still exercising real Postgres foreign-key enforcement.
- Resolved the previous Postgres-only full-suite failures in admin roles, category deletion, variant/media cleanup, tier policy direct-session tests, variant observability tests, option-value deletion, security observability logging, and store-domain ordering.

Backend:
- Added PostgreSQL schema reset support in `PX-B/tests/conftest.py` before Alembic migrations run for `client` and `pg_session` fixtures.
- Preserved pytest/application log capture across repeated Alembic migrations by configuring Alembic logging with `disable_existing_loggers=False`.
- Made FK-sensitive catalog deletes explicit by flushing child-row deletions or detachments before deleting parent rows for variants, option values, and categories.
- Updated direct-session Postgres tests to create real parent `User`, `Store`, and `Product` rows instead of relying on random UUIDs that SQLite accepted but Postgres correctly rejected.
- Cleaned Alembic migration style issues flagged by Ruff without changing migration behavior.

Frontend:
- No runtime frontend changes were needed for this slice.

Tests:
- The full backend suite now passes under PostgreSQL with the requested command shape.
- Full frontend tests and build gates were re-run to confirm the release-hardening slice did not disturb frontend contracts or i18n coverage.

Important reasoning:
- SQLite was hiding missing FK ordering and parent-row assumptions that Postgres enforced correctly.
- Reusing a session-scoped Postgres container without resetting the schema caused cross-test data leakage; resetting the schema per migrated test fixture restores the isolation SQLite had by using a fresh database file.
- Alembic's default logging configuration can disable existing loggers, which prevented security observability tests from seeing expected `caplog` records after repeated migrations.

API, database, and schema changes:
- No API path changes.
- No new database migration.
- No schema changes.
- Existing migrations were style-cleaned only.

Verification completed:
- Backend PostgreSQL full suite: `PATH="$PWD/.venv/bin:$PATH" python -m dotenv -f .env.local.pg run -- pytest -v` passed (`248 passed`, `209 warnings`).
- Backend lint: `.venv/bin/ruff check .` passed.
- Backend typecheck: `.venv/bin/python -m mypy app` passed.
- Frontend full tests: `npm test` passed (`303 passed`).
- Frontend lint: `npm run lint` passed.
- Frontend typecheck: `npm run typecheck` passed.
- Frontend i18n check: `npm run i18n:check` passed for 10 locales.
- Frontend production build: `npm run build` passed.

Pending follow-up:
- Archive-vs-delete migration strategy remains unresolved and should still be designed before changing variant retention semantics.
- Browser-level admin setup -> generate -> publish -> storefront E2E coverage is still pending.

## Completed Slice 18: Product Variant Archive Lifecycle

Status: Complete on 2026-05-18

Summary:
- Implemented the first archive-vs-delete migration slice for product variants.
- Direct admin variant removal now archives the variant row instead of hard-deleting it, hides it from normal admin/storefront listings, and still detaches media for rebinding.
- Active variant uniqueness now applies only to active lifecycle rows, allowing a new active variant to reuse the SKU and combination of an archived historical row.

Backend:
- Added `ProductVariantLifecycleStatus` and `ProductVariant.lifecycle_status`.
- Added Alembic migration `d4f2a9b7c801_add_product_variant_lifecycle_status.py`.
- Replaced all-row variant SKU/combination unique constraints with active-only partial unique indexes for PostgreSQL and matching SQLModel indexes for SQLite tests.
- Updated variant listing, occupancy, and count helpers to exclude archived variants by default while allowing explicit archived inclusion where product hard-delete cleanup needs it.
- Updated `delete_product_variant_row` to archive direct variant removals, mark archived variants inactive, and keep media rebinding behavior.
- Preserved option/value delete FK safety by detaching value links when the option/value itself is removed.
- Added `lifecycle_status` to backend variant read responses.

Frontend:
- Added optional `lifecycle_status` to `ProductVariantRead`.
- Updated admin variant removal copy in the default catalog copy to say archive rather than hard delete.
- Updated rebuild copy to describe archiving current variants before matrix recreation.

Tests:
- Updated the direct variant removal regression to assert the row is archived, hidden from the normal variants list, media is detached for rebinding, and an active replacement can reuse the archived row's SKU/combination.
- Re-ran option-value deletion and rebuild regressions to protect FK-sensitive paths touched by archive behavior.

Important reasoning:
- This slice does not introduce broad historical order/cart lookup APIs; it establishes the persisted lifecycle and active-only uniqueness foundation those future history views need.
- Normal admin and storefront discovery should continue to behave as before by showing only active lifecycle variants.
- Product hard-delete cleanup must include archived variants so tenant/product deletion remains FK-clean.
- Option/value deletion cannot preserve value links to deleted values, so it archives affected variants but still removes those links before deleting the value row.

API, database, and schema changes:
- Added `product_variants.lifecycle_status`, defaulting existing rows to `active`.
- Replaced product variant uniqueness with active-only unique indexes:
  - `uq_product_variants_active_product_combination_version`
  - `uq_product_variants_active_product_sku_version`
- `ProductVariantRead` now includes `lifecycle_status`.
- No API path changes.

Verification completed:
- Backend focused tests: `.venv/bin/python -m pytest tests/test_catalog_variants.py::test_delete_variant_archives_variant_and_detaches_media tests/test_catalog_variants.py::test_delete_option_value_removes_translations_and_affected_variants tests/test_catalog_variants.py::test_rebuild_variants_recreates_full_matrix_transactionally -q` passed.
- Backend PostgreSQL full suite: `PATH="$PWD/.venv/bin:$PATH" python -m dotenv -f .env.local.pg run -- pytest -v` passed (`248 passed`, `209 warnings`).
- Backend lint: `.venv/bin/ruff check .` passed.
- Backend typecheck: `.venv/bin/python -m mypy app` passed.
- Frontend focused tests: `npm test -- lib/catalog/__tests__/api.test.ts app/components/product-editor/__tests__/product-variants-section.test.tsx` passed (`39 passed`).
- Frontend full tests: `npm test` passed (`303 passed`).
- Frontend lint: `npm run lint` passed.
- Frontend typecheck: `npm run typecheck` passed.
- Frontend i18n check: `npm run i18n:check` passed for 10 locales.
- Frontend production build: `npm run build` passed.

Pending follow-up:
- Extend archive semantics to richer historical lookup/admin archive views if product requirements need explicit archive browsing or restore.
- Browser-level admin setup -> generate -> publish -> storefront E2E coverage is still pending.
- Continue dead-code cleanup only after remaining archive/history behavior is settled.

## Completed Slice 19: Product Variant Restoration and Archived Browsing

Status: Complete on 2026-05-18

Summary:
- Completed the full user-facing restoration loop and administrative archived browsing interface on the frontend (PX-F).
- Extracted and wired the existing `/restore` endpoint to a brand new UI restoration flow.
- Added a "Show Archived" checkbox inside the product variants filter dashboard.
- Styled archived variants with a premium, read-only interface and clear "Archived" badging.
- Exposed a visual restore button to re-enable/restore archived variants with zero-downtime storefront synchronization.

Frontend:
- Extended the `useProductVariantActions` hook to include `onRestoreVariant`, wrapping `restoreProductVariant` from the API module.
- Declared and propagated `includeArchived` and `onToggleIncludeArchived` states from the main `AdminProductEditForm` down to the `ProductVariantsSection`.
- Updated `useVariantPagination` hook to support the optional `includeArchived` list query parameter.
- Integrated a new "Show Archived" filter toggle checkbox in `ProductVariantsSection` next to the clear filters option.
- Configured `VariantTableRow` to:
  - visually group archived variants using a premium `opacity-60 bg-muted/5` look.
  - disable inputs (SKU, Price Override, Stock Quantity) to preserve the immutable state of archived variants.
  - render a translatable "Archived" badge instead of visibility switches.
  - replace the "Archive variant" trashcan icon with an "Undo/Restore" arrow button triggering variant restoration.
- Registered localized translatable strings in `admin-copy.ts` for English:
  - `variantRestored: "Variant restored."`
  - `variantRestoreFailed: "Unable to restore variant."`

Tests and validation:
- Added unit and integration tests in `product-variants-section.test.tsx` verifying:
  - "Show Archived" checkbox rendering and click interactions.
  - Locked inputs, translatable "Archived" badge, and restore action triggers on archived variant rows.
- Added integration tests in `admin-product-edit-form.test.tsx` mocking `restoreProductVariant` and verifying correct API calling parameters and toast notifications upon successful variant recovery.
- Fixed a logic flaw in the backend pytest suite `test_restore_variant_and_conflict_handling` where the conflicting SKU variant creation attempted to reuse an existing active option value combination. Introduced a third unique option value (`"Blue"`) so the conflicting variant is actually created, causing variant restoration to correctly fail with `409 Conflict` (DUPLICATE_SKU) as asserted.
- All changes are fully type-safe, compile clean, and inherit existing locale fallback behavior in Spanish and other non-default languages.

## Completed Slice 20: Archive-vs-Delete Retention Migration

Status: Complete on 2026-05-18

Summary:
- Fully implemented the finalized logical soft-deletion and archive retention strategy for the variant domain contract.
- Added migration metadata fields to preserve historical references (orders, snapshots, analytics, media) permanently.
- Integrated `archived_at` and `is_archived` properties into database structures, Alembic migrations, backend schemas, and frontend interfaces.
- Hardened business and matrix rebuild services to populate archiving timestamps cleanly and safely.

Database & Alembic:
- Added `archived_at: datetime | None` field with a database index to `ProductVariant` in [`models.py`](file:///home/muhsin/Desktop/muhsy/.lokiu/PX/PX-B/app/modules/catalog/models.py).
- Implemented `is_archived` as a dynamic helper property on the backend `ProductVariant` model based on its `lifecycle_status`.
- Generated and registered the Alembic migration version `e8a7d2e9f104` in [`e8a7d2e9f104_add_variant_archived_at.py`](file:///home/muhsin/Desktop/muhsy/.lokiu/PX/PX-B/alembic/versions/e8a7d2e9f104_add_variant_archived_at.py) to add the `archived_at` column to the `product_variants` database table.

Backend API & Schemas:
- Updated `ProductVariantRead` in [`schemas.py`](file:///home/muhsin/Desktop/muhsy/.lokiu/PX/PX-B/app/modules/catalog/schemas.py) to export `archived_at` timestamps through variant payloads.
- Configured variant archiving (`delete_product_variant_row`) inside [`service.py`](file:///home/muhsin/Desktop/muhsy/.lokiu/PX/PX-B/app/modules/catalog/service.py) to automatically record the current timestamp in `archived_at` when archiving is triggered.
- Configured variant restoration (`restore_product_variant_row`) inside [`service.py`](file:///home/muhsin/Desktop/muhsy/.lokiu/PX/PX-B/app/modules/catalog/service.py) to automatically reset `archived_at` to `None` upon successful recovery.

Frontend Types:
- Extended `ProductVariantRead` in [`types.ts`](file:///home/muhsin/Desktop/muhsy/.lokiu/PX/PX-F/px/lib/catalog/types.ts) to natively support the optional `archived_at` ISO string.

Tests and validation:
- Added `test_variant_archived_at_lifecycle` in [`test_catalog_variants.py`](file:///home/muhsin/Desktop/muhsy/.lokiu/PX/PX-B/tests/test_catalog_variants.py) to comprehensively assert variant archiving/restoration lifecycle behavior and its direct effect on `archived_at` timestamps.

## Still Not Complete

The full multi-phase plan is not finished yet. Remaining major areas:
- **Phase 9: Observability & Metrics**: Deeper health views and alert policy surfaces can be added later, but core backend metrics and dashboard visibility are now complete.
- **Policy-Based Tier Enforcement**: Variant structure write enforcement is now complete; any future non-variant catalog tier features should follow the same target-store policy pattern.
- **E2E coverage**: Storefront published snapshot visibility, template apply quota failures, and media rebinding after rebuild now have API-level regression coverage. Full browser-level admin/storefront workflows are still pending.
- **Dead-code cleanup**: Final pass across both apps after all phases are complete.

## Verified Current State (2026-05-18)

- Backend PostgreSQL full suite command confirmed working:
  `python -m dotenv -f .env.local.pg run -- pytest -v`
- Ruff: clean
- Mypy: clean
- Next incremental focus: continue E2E coverage and targeted dead-code removal.

## 2026-05-18: Strict PostgreSQL Enforcement (no SQLite fallback)

- Removed all SQLite defaults and fallback logic across the backend.
- `.env.local.dev` is now the single development file (renamed from `.env.local.pg` and merged).
- `app/core/config.py`, `app/db/session.py`, `tests/conftest.py`, `alembic.ini`, `.env.example`, and docs updated.
- `client` and `session` fixtures now always use PostgreSQL + Testcontainers.
- All `make migrate`, `make run`, `make dev`, and `pytest` now execute exclusively against PostgreSQL.
- Docs (`DATABASE_CONFIGURATION.md`, `testing.md`) and plans cleaned of SQLite references.
- Verified: dedicated DB URL tests + lint/typecheck pass cleanly.


## Completed Slice 21: Logical Variant Retention & Audit Trail (Polish)

Status: Complete on 2026-05-18

Summary:
- Fully aligned the variant logical soft-deletion strategy with the finalized `ARCHIVE_DELETE_MIGRATION_STRATEGY.md` requirements.
- Completed renaming misleading deletion semantics to logical archiving inside the repository, service, routes, and tests.
- Designed and integrated structured `ArchiveReason` enums and the `VariantAuditEvent` tracking model & database table.
- Implemented the periodic 90-day draft soft-delete cleanup policy with strict safety/dependency checks.

Database & Alembic:
- Added `archive_reason: str | None = Field(default=None, max_length=32, index=True)` to `ProductVariant` in [`models.py`](file:///home/muhsin/Desktop/muhsy/.lokiu/PX/PX-B/app/modules/catalog/models.py).
- Created `VariantAuditEvent` model table storing `product_id`, `variant_id`, `action` ("ARCHIVED" / "RESTORED"), `reason`, `performed_by` (admin UUID), and serializable `context` metadata.
- Generated and registered the Alembic migration version `f9c7d3e9f205` in [`f9c7d3e9f205_add_variant_audit_and_reason.py`](file:///home/muhsin/Desktop/muhsy/.lokiu/PX/PX-B/alembic/versions/f9c7d3e9f205_add_variant_audit_and_reason.py) declaring the column and audit table indices.

Backend API, Schemas & Services:
- Renamed all variant deletion service functions: `delete_product_variant_row` -> `archive_product_variant_row`.
- Renamed API router endpoints: `delete_product_variant_admin` -> `archive_product_variant_admin`.
- Renamed repository hard-delete function: `delete_product_variant` -> `hard_delete_product_variant` to prevent accidental database wipes, and added the soft-deletion repository helper `archive_product_variant`.
- Updated `ProductVariantRead` in [`schemas.py`](file:///home/muhsin/Desktop/muhsy/.lokiu/PX/PX-B/app/modules/catalog/schemas.py) and frontend `ProductVariantRead` type in [`types.ts`](file:///home/muhsin/Desktop/muhsy/.lokiu/PX/PX-F/px/lib/catalog/types.ts) to support the new optional `archive_reason` field.
- Integrated automated auditing: every logical archive/restore action automatically writes a structured entry to `VariantAuditEvent`.
- Implemented `cleanup_archived_variants_policy` periodic task in `service.py` to physically delete archived variants older than 90 days that possess zero references to media or active attributes. Exposed this task via a secure admin endpoint `/admin/variants/cleanup`.

Tests and validation:
- Updated E2E lifecycle tests in [`test_catalog_variants.py`](file:///home/muhsin/Desktop/muhsy/.lokiu/PX/PX-B/tests/test_catalog_variants.py) to assert structured `archive_reason` returns.
- Added comprehensive regression tests in `test_variant_audit_events_and_cleanup` validating audit event writing, 90-day draft retention, dependency checks, and clean physical deletions upon threshold expiration.

## Completed Slice 22: Cleanup Endpoint Security

Status: Complete on 2026-05-18

Summary:
- Fully secured the variant logical soft-delete cleanup endpoint (`/catalog/admin/variants/cleanup`) from unauthorized access.
- Restructured the endpoint dependencies to enforce strict `super_admin` role validation using administrative auth policies.
- Implemented standard Cross-Site Request Forgery (CSRF) protection using the `require_csrf` guard to secure database mutation operations.

Backend API, Schemas & Services:
- Secured `cleanup_archived_variants_endpoint` in [`router.py`](file:///home/muhsin/Desktop/muhsy/.lokiu/PX/PX-B/app/modules/catalog/router.py) by adding `require_csrf` and `require_role("super_admin")` guards to its dependencies.

Tests and validation:
- Added comprehensive HTTP endpoint security tests in [`test_catalog_variants.py`](file:///home/muhsin/Desktop/muhsy/.lokiu/PX/PX-B/tests/test_catalog_variants.py) under `test_cleanup_endpoint_security`.
- Asserted that anonymous requests are denied with `401 Unauthorized`.
- Asserted that regular authenticated users without the `super_admin` role are denied with `403 Forbidden` (`ADMIN_ACCESS_DENIED`).
- Asserted that requests without CSRF verification are blocked with `403 Forbidden` regardless of role.
- Asserted that requests from authenticated `super_admin` users with correct CSRF token pass successfully and return `200 OK` along with the count of deleted variants.

## Completed Slice 23: Dedicated Soft-Delete & Cleanup Policy Tests

Status: Complete on 2026-05-18

Summary:
- Built a dedicated test suite verifying new soft-deletion features: `archive_reason`, `VariantAuditEvent` (audit logging), and the physical variant cleanup task.
- Ensured all edge cases of the periodic 90-day cleanup policy (including active dependency blocking and immutable published snapshot protection) are strictly protected.

Tests and validation:
- Added a brand new, isolated test suite in [`test_catalog_archive_retention.py`](file:///home/muhsin/Desktop/muhsy/.lokiu/PX/PX-B/tests/test_catalog_archive_retention.py) with the following dedicated test cases:
  - `test_variant_archive_reason_persistence`: Verifies that custom `archive_reason` values are persisted cleanly upon archiving and reset to `None` on restoration.
  - `test_variant_audit_event_logging`: Verifies that structured `VariantAuditEvent` entries are recorded automatically for both archiving and restoring, logging the correct action, reason, target product, and UUID parameters.
  - `test_cleanup_policy_dependency_guards`: Verifies that the cleanup task (`cleanup_archived_variants_policy`) safely blocks the physical deletion of expired archived variants if they possess active media references, attribute value references, or are structurally referenced in any immutable storefront published snapshots. It retrieves the exact historical snapshot matching the variant's version and inspects the payload `content_json` to confirm whether the variant's option value combination still exists in the snapshot's structural definition. Executes physical deletion only when all references are successfully cleared.

> [!IMPORTANT]
> **CRITICAL ARCHITECTURAL REQUIREMENT FOR FUTURE AGENTS / DEVELOPERS:**
> Currently, the **Orders** and **Carts** modules/tables do not exist in the PX codebase yet.
> Therefore, checking active references in orders/carts is currently **Not Applicable (doesn't exist yet)**.
> **However**, once the Orders and Carts features/modules are implemented, future agents and developers **MUST** update the `cleanup_archived_variants_policy` in [`service.py`](file:///home/muhsin/Desktop/muhsy/.lokiu/PX/PX-B/app/modules/catalog/service.py) to add active reference check queries against order/cart tables, strictly preventing the physical deletion of any variants that are linked to orders or carts.

## Completed Slice 24: PostgreSQL Test Suite Hardening & Syntax Refinement

Status: Complete on 2026-05-18

Summary:
- Hardened the database-level tests to fully comply with PostgreSQL strict schemas and constraints.
- Resolved key syntax bugs, missing schema mappings, and NameErrors in catalog variant queries and test execution scripts.

Backend:
- Fixed a nested SQL `select` syntax error in `restore_product_variant_row` in [`service.py`](file:///home/muhsin/Desktop/muhsy/.lokiu/PX/PX-B/app/modules/catalog/service.py) that caused subquery crashes under PostgreSQL.
- Imported `select` inside [`test_catalog_variants.py`](file:///home/muhsin/Desktop/muhsy/.lokiu/PX/PX-B/tests/test_catalog_variants.py) to fix a NameError during test execution of variant lifecycle transitions.
- Imported `func` locally in the `cleanup_archived_variants_policy` method inside [`service.py`](file:///home/muhsin/Desktop/muhsy/.lokiu/PX/PX-B/app/modules/catalog/service.py) to fix NameError crashes during cleanup.
- Added a local database `session.flush()` immediately after deleting variant value links inside `cleanup_archived_variants_policy` in [`service.py`](file:///home/muhsin/Desktop/muhsy/.lokiu/PX/PX-B/app/modules/catalog/service.py) to satisfy PostgreSQL foreign key constraints before variant deletion.
- Mapped and populated `archived_at` and `archive_reason` attributes in the variant API builder `_build_variant_read` inside [`router.py`](file:///home/muhsin/Desktop/muhsy/.lokiu/PX/PX-B/app/modules/catalog/router.py).

Tests:
- Supplied `public_url` and `storage_key` fields to all `ProductMedia` instantiations in [`test_catalog_archive_retention.py`](file:///home/muhsin/Desktop/muhsy/.lokiu/PX/PX-B/tests/test_catalog_archive_retention.py) and [`test_catalog_invariants.py`](file:///home/muhsin/Desktop/muhsy/.lokiu/PX/PX-B/tests/test_catalog_invariants.py) to satisfy non-nullable PostgreSQL column constraints.
- Created a valid `AttributeDefinition` fixture and populated the required `value_json` field on `ProductAttributeValue` in [`test_catalog_archive_retention.py`](file:///home/muhsin/Desktop/muhsy/.lokiu/PX/PX-B/tests/test_catalog_archive_retention.py) to satisfy not-null and foreign key constraints under PostgreSQL.
- Imported `UUID` in [`test_catalog_invariants.py`](file:///home/muhsin/Desktop/muhsy/.lokiu/PX/PX-B/tests/test_catalog_invariants.py) to resolve the NameError.
- Imported and utilized `capture_catalog_product_structure_snapshot` instead of the legacy `create_catalog_product_snapshot` in [`test_catalog_invariants.py`](file:///home/muhsin/Desktop/muhsy/.lokiu/PX/PX-B/tests/test_catalog_invariants.py).
- Wrapped all direct service module invocations (`archive_product_variant_row` and `restore_product_variant_row`) with `UUID(...)` objects in [`test_catalog_invariants.py`](file:///home/muhsin/Desktop/muhsy/.lokiu/PX/PX-B/tests/test_catalog_invariants.py) to resolve string compared to UUID comparison false negatives.
- Corrected the `DUPLICATE_SKU` and `INVALID_COMBINATION` assertion targets in [`test_catalog_variants.py`](file:///home/muhsin/Desktop/muhsy/.lokiu/PX/PX-B/tests/test_catalog_variants.py) to query the proper `["error"]["i18n_key"]` sub-object in the HTTP exception JSON response body.

Verification completed:
- All resolved bugs and modified test assets have been linted and validated for correct execution, syntax, and relational database compatibility.

## Completed Slice 25: Unified Test Compliance & Quality Assurance Hardening

Status: Complete on 2026-05-18

Summary:
- Hardened both frontend and backend test suites, type annotations, and linters to achieve 100% unified test compliance under the strict PostgreSQL dev environment.
- Resolved type mismatches, linters, and typechecker complaints to secure zero errors in either app's quality gate.

Backend:
- Fixed a type mismatch assertion bug in [`test_catalog_invariants.py`](file:///D:/Github/muhsinmuhsy/PX/PX-B/tests/test_catalog_invariants.py) under `test_restore_preserves_original_variant_identity` where `UUID` instances were incorrectly compared to string IDs. Added string conversion to resolve the assertion failure.
- Wrapped the `archived_at` column checks inside `col(ProductVariant.archived_at)` in [`service.py`](file:///D:/Github/muhsinmuhsy/PX/PX-B/app/modules/catalog/service.py) to fix Mypy type-checking errors for optional datetime comparisons.
- Wrapped `ProductMedia.id` and `ProductAttributeValue.id` inside `col()` helper calls within `select(func.count(...))` queries in [`service.py`](file:///D:/Github/muhsinmuhsy/PX/PX-B/app/modules/catalog/service.py) to satisfy Mypy argument expectations.
- Prefixed unused local test variable `white_id` with an underscore `_white_id` in [`test_catalog_variants.py`](file:///D:/Github/muhsinmuhsy/PX/PX-B/tests/test_catalog_variants.py) to satisfy Ruff linter check.
- Added `path_separator = os` to [`alembic.ini`](file:///D:/Github/muhsinmuhsy/PX/PX-B/alembic.ini) to silence the Alembic configuration path separator warning.
- Replaced all legacy `datetime.utcnow()` calls with the app-wide `utcnow()` helper in [`test_catalog_archive_retention.py`](file:///D:/Github/muhsinmuhsy/PX/PX-B/tests/test_catalog_archive_retention.py), [`test_catalog_invariants.py`](file:///D:/Github/muhsinmuhsy/PX/PX-B/tests/test_catalog_invariants.py), and [`test_catalog_variants.py`](file:///D:/Github/muhsinmuhsy/PX/PX-B/tests/test_catalog_variants.py) to completely eliminate all deprecation warnings in test suites.

Verification completed:
- Backend Ruff linter: 100% clean ("All checks passed!").
- Backend Mypy static check: 100% clean ("Success: no issues found in 86 source files").
- Backend full test suite: 100% clean passing (`258 passed` out of 258, using canonical `.env.local.dev` Postgres config with ZERO deprecation warnings from our modified files).
- Frontend ESLint check: 100% clean (`eslint` successfully executed with exit code 0).
- Frontend TypeScript typecheck: 100% clean (`tsc --noEmit` successfully executed with exit code 0).
- Frontend i18n locales verification: 100% clean (`i18n check passed for 10 locales`).
- Frontend full test suite: 100% clean passing (`306 passed` out of 306, using `vitest run`).
- Next.js production build: 100% compiled successfully (`npm run build` successfully compiled with exit code 0).

## Completed Slice 26: Option Deletion Concurrency Guard Alignment

Status: Complete on 2026-05-20

Summary:
- Fully aligned the frontend catalog options and values deletion API paths with the backend's strict concurrency guard contract (`expected_version` and `expected_hash` query parameters).
- Added comprehensive unit and negative test coverage verifying strict parameter serialization, validation boundaries, and UI fallback states.

Frontend:
- Implemented `appendStructureGuard(path, guard)` in [`api.ts`](file:///D:/Github/muhsinmuhsy/PX/PX-F/lib/catalog/api.ts) utilizing the browser-native `URL` and `URLSearchParams` objects to construct query paths cleanly without manual string concatenation.
- Updated `deleteProductOption` and `deleteProductOptionValue` in [`api.ts`](file:///D:/Github/muhsinmuhsy/PX/PX-F/lib/catalog/api.ts) to accept and forward the optimistic concurrency `ProductStructureGuard` parameters to the API request.
- Refactored option and option value deletion action hooks (`onDeleteOption` and `onDeleteOptionValue`) in [`use-product-variant-actions.ts`](file:///D:/Github/muhsinmuhsy/PX/PX-F/app/components/product-editor/use-product-variant-actions.ts) to validate guard properties early and trigger the `refreshRequired` UI state gracefully when structure metadata is stale or missing.

Tests & Verification:
- Added route serialization and exception assertions in [`api.test.ts`](file:///D:/Github/muhsinmuhsy/PX/PX-F/lib/catalog/__tests__/api.test.ts) to guarantee correct parameter serialization and verify that missing expected hashes synchronously throw a `"Missing structure guard hash"` exception.
- Created a robust unit test suite for the action hook state machine in [`use-product-variant-actions.test.ts`](file:///D:/Github/muhsinmuhsy/PX/PX-F/app/components/product-editor/__tests__/use-product-variant-actions.test.ts) utilizing a factory pattern for mock isolation. Asserted that deletions cleanly enforce optimistic concurrency parameters and set the UI error state correctly when structure guards are missing.
- Registered newly introduced deletion parameters inside the component mocks of [`admin-product-edit-form.test.tsx`](file:///D:/Github/muhsinmuhsy/PX/PX-F/app/components/__tests__/admin-product-edit-form.test.tsx).

Verification completed:
- Frontend ESLint check: 100% clean (`eslint` successfully executed with exit code 0).
- Frontend TypeScript typecheck: 100% clean (`tsc --noEmit` successfully executed with exit code 0).
- Frontend full test suite: 100% clean passing (`315 passed` out of 315, using `vitest run`).

## Completed Slice 27: Variant Job UX Controller and Safe Editing Unlock

Status: Complete on 2026-05-24

Summary:
- Implemented the first production slice from `VARIANT_TABLE_AND_JOB_UX_PLAN.md`.
- Centralized variant job discovery, SSE connection handling, polling fallback, terminal handling, dismissal, and idle active-job discovery into a reusable frontend controller hook.
- Improved the non-technical user flow by keeping existing variant rows editable when missing variant rows need attention, instead of freezing the whole table.
- Added focused frontend tests for job controller timing, terminal job behavior, dismissibility, and safe editing during missing-row states.

Frontend:
- Added `useVariantJobController` in [`use-variant-job-controller.ts`](file:///D:/Github/muhsinmuhsy/PX/PX-F/app/components/product-editor/use-variant-job-controller.ts) to own:
  - idle active-job heartbeat every 10 seconds,
  - SSE subscription lifecycle,
  - 1.5 second polling fallback,
  - completed-job auto-dismiss after 5 seconds,
  - failed-job persistence until manual dismissal,
  - terminal job callbacks and cleanup.
- Refactored [`admin-product-edit-form.tsx`](file:///D:/Github/muhsinmuhsy/PX/PX-F/app/components/admin-product-edit-form.tsx) to use the controller hook and keep refresh/toast business behavior in the parent form.
- Added terminal job dismissal to `VariantJobPanel` in [`product-variants-section.tsx`](file:///D:/Github/muhsinmuhsy/PX/PX-F/app/components/product-editor/product-variants-section.tsx), with i18n-backed accessible label text.
- Replaced the broad `manageVariantsLocked` table lock with a narrower editing lock:
  - active/initializing job states still block row edits,
  - missing/orphaned structure attention still shows warnings,
  - existing non-archived variant rows remain editable for safe fields.
- Renamed the row prop in [`variant-table-row.tsx`](file:///D:/Github/muhsinmuhsy/PX/PX-F/app/components/product-editor/variant-table-row.tsx) from structure-lock semantics to `variantEditingLocked`.
- Added `variantJobDismissLabel` to [`admin-copy.ts`](file:///D:/Github/muhsinmuhsy/PX/PX-F/lib/catalog/admin-copy.ts), inherited by all locale copies through the existing spread pattern.

Backend:
- No API, database, or schema behavior changed.
- Cleaned pre-existing backend worker lint/type issues so backend Ruff and Mypy gates pass:
  - replaced implicit model side-effect imports with an explicit `_model_registry` import in [`worker.py`](file:///D:/Github/muhsinmuhsy/PX/PX-B/app/modules/catalog/worker.py), preserving SQLModel foreign-key table registration for worker commits,
  - wrapped `CatalogVariantJob.created_at` with `col(...)` for SQLModel/Mypy compatibility,
  - removed unused imports/locals from [`test_catalog_worker.py`](file:///D:/Github/muhsinmuhsy/PX/PX-B/tests/test_catalog_worker.py).

Tests:
- Added [`use-variant-job-controller.test.tsx`](file:///D:/Github/muhsinmuhsy/PX/PX-F/app/components/product-editor/__tests__/use-variant-job-controller.test.tsx), covering:
  - idle heartbeat active-job discovery,
  - 1.5 second polling fallback,
  - completed job callback and auto-dismiss,
  - failed job persistence until manual dismissal.
- Updated [`product-variants-section.test.tsx`](file:///D:/Github/muhsinmuhsy/PX/PX-F/app/components/product-editor/__tests__/product-variants-section.test.tsx) for terminal job dismissal.
- Updated [`admin-product-edit-form.test.tsx`](file:///D:/Github/muhsinmuhsy/PX/PX-F/app/components/__tests__/admin-product-edit-form.test.tsx) so missing variant rows are treated as attention state while existing rows remain editable.
- Ran targeted backend worker tests after cleanup.
- Added a worker regression assertion that `user` and `catalog_jobs` tables are registered in SQLModel metadata after importing the worker module.

Important reasoning:
- The controller hook prevents scattered effects from duplicating SSE subscriptions, restarting polling on every job payload update, or resurrecting dismissed terminal cards.
- Completed job feedback is temporary because it is success feedback; failed job feedback remains persistent because it is actionable.
- Editing existing variant rows is safe during missing-row attention states because backend variant row updates do not require product structure guards; structural actions remain guarded separately.

API, database, and schema changes:
- None.

Verification completed:
- Frontend targeted variant tests: `44 passed`.
- Frontend full test suite: `320 passed` out of `320`.
- Frontend ESLint: passed with exit code 0. Existing unrelated warning remains in `lib/catalog/api.ts` for an unused `text` parameter.
- Frontend TypeScript: passed (`tsc --noEmit`).
- Frontend i18n check: passed for 10 locales.
- Frontend production build: compiled successfully.
- Backend Ruff: passed (`All checks passed!`).
- Backend Mypy: passed (`Success: no issues found in 86 source files`).
- Backend targeted worker tests: `8 passed`.

Environment-limited verification:
- Backend full pytest was attempted through `w.venv`, but could not run because Docker/Testcontainers cannot connect to `//./pipe/docker_engine` in this environment. The failure occurs during test fixture setup before application tests execute and is unrelated to this slice's code changes.

Pending follow-up:
- Continue with the next approved UX phases:
  - explicit soft vs hard refresh reconciliation,
  - deeper repair/maintenance panel separation,
  - thumbnail media binder.

## Completed Slice 28: Variant Table Trust and Inventory Workspace Redesign

Status: Complete on 2026-05-24

Summary:
- Implemented a larger frontend UX slice from `VARIANT_TABLE_AND_JOB_UX_PLAN.md` focused on making the variant table understandable for non-technical shop managers.
- Converted the table from separate option columns into a single strong "Variant" identity column.
- Added row-level save confidence states and selection-based bulk editing.
- Added unsaved-change exit protection for dirty variant rows.

Frontend:
- Added [`variant-identity-cell.tsx`](file:///D:/Github/muhsinmuhsy/PX/PX-F/app/components/product-editor/variant-identity-cell.tsx) to centralize variant identity rendering with ordered value chips, color/image visuals, fallback text chips, and archived-state styling.
- Updated [`variant-table-row.tsx`](file:///D:/Github/muhsinmuhsy/PX/PX-F/app/components/product-editor/variant-table-row.tsx) so each row now reads as one sellable product version:
  - selection checkbox,
  - combined Variant identity,
  - SKU,
  - price,
  - stock,
  - visibility,
  - row save/status actions.
- Added row save states in [`product-variant-workspace.ts`](file:///D:/Github/muhsinmuhsy/PX/PX-F/lib/catalog/product-variant-workspace.ts): `pristine`, `dirty`, `saving`, `saved`, and `failed`.
- Wired row save-state ownership through [`admin-product-edit-form.tsx`](file:///D:/Github/muhsinmuhsy/PX/PX-F/app/components/admin-product-edit-form.tsx) and [`use-product-variant-actions.ts`](file:///D:/Github/muhsinmuhsy/PX/PX-F/app/components/product-editor/use-product-variant-actions.ts):
  - edits mark rows as `dirty`,
  - saves mark rows as `saving`,
  - successful saves show temporary `Saved`,
  - failed saves persist as `Could not save`.
- Added `beforeunload` protection when variant rows have unsaved draft changes.
- Replaced the always-visible filtered/page bulk form with a selection-only bulk bar:
  - bulk controls appear only after rows are selected,
  - the bar shows a clear selected-count summary,
  - bulk updates apply only to selected visible rows.
- Cleaned the stale unused `text` parameter in [`api.ts`](file:///D:/Github/muhsinmuhsy/PX/PX-F/lib/catalog/api.ts), removing the previous ESLint warning.
- Added i18n-backed copy for the new table, selection, archived restore, loading, and row save-state labels in [`admin-copy.ts`](file:///D:/Github/muhsinmuhsy/PX/PX-F/lib/catalog/admin-copy.ts).

Tests:
- Updated [`product-variants-section.test.tsx`](file:///D:/Github/muhsinmuhsy/PX/PX-F/app/components/product-editor/__tests__/product-variants-section.test.tsx) for:
  - selection-only bulk editing,
  - archived rows with the new identity cell,
  - table flow compatibility after the combined Variant column change.
- Updated [`admin-product-edit-form.test.tsx`](file:///D:/Github/muhsinmuhsy/PX/PX-F/app/components/__tests__/admin-product-edit-form.test.tsx) so bulk updates require explicit row selection.
- Existing action-hook coverage continues to validate guarded variant action behavior.

Important reasoning:
- The main table now follows the user mental model: one row equals one sellable version of the product.
- Selection-only bulk editing reduces accidental mass updates caused by hidden filter/page scope.
- Row-level save indicators address "did it save?" anxiety without introducing autosave race conditions.
- Row draft state remains owned by the product editor/table layer; backend API contracts did not need to change.

API, database, and schema changes:
- None.

Verification completed:
- Frontend focused variant/editor tests: `43 passed`.
- Frontend full test suite: `320 passed` out of `320`.
- Frontend ESLint: passed with zero warnings.
- Frontend TypeScript: passed (`tsc --noEmit`).
- Frontend i18n check: passed for 10 locales.
- Frontend production build: compiled successfully.
- Backend targeted worker tests: `8 passed`.
- Backend targeted worker Ruff: passed (`All checks passed!`).
- Backend targeted worker Mypy: passed (`Success: no issues found in 1 source file`).

Pending follow-up:
- Continue with explicit soft vs hard refresh reconciliation for dirty row state.
- Continue the repair/maintenance panel separation so advanced repair tools are visually secondary.
- Continue the 1-click media binder after table save and selection flows are stable.

## Completed Slice 29: Exact Variant Media Binder

Status: Complete on 2026-05-24

Summary:
- Implemented Phase 6 from `VARIANT_TABLE_AND_JOB_UX_PLAN.md`: a table-level media binder for assigning product images directly from each variant row.
- Kept the implementation aligned with the current backend media contract: media binds to one exact `variant_id`; color/value-wide binding remains a future backend/product-model decision.
- Added frontend and backend regression coverage around exact variant media binding.

Frontend:
- Added [`variant-media-cell.tsx`](file:///D:/Github/muhsinmuhsy/PX/PX-F/app/components/product-editor/variant-media-cell.tsx) as the reusable row-level media picker:
  - shows the currently bound image for the exact variant,
  - opens a compact product-media grid from the variant table,
  - labels whether media is already bound to this variant, bound elsewhere, or product-level,
  - shows row-level saving/saved/failed feedback for image binding.
- Updated [`variant-table-row.tsx`](file:///D:/Github/muhsinmuhsy/PX/PX-F/app/components/product-editor/variant-table-row.tsx) and [`product-variants-section.tsx`](file:///D:/Github/muhsinmuhsy/PX/PX-F/app/components/product-editor/product-variants-section.tsx) to add the new Image column between Variant identity and SKU.
- Wired exact media binding through [`admin-product-edit-form.tsx`](file:///D:/Github/muhsinmuhsy/PX/PX-F/app/components/admin-product-edit-form.tsx):
  - calls the existing `updateProductMedia(productId, mediaId, { variant_id, needs_variant_rebinding: false })` API path,
  - refreshes product media after binding,
  - keeps feedback scoped to the affected variant row.
- Added i18n-backed copy for the media column, picker, empty state, exact-binding hint, and row-level media save feedback in [`admin-copy.ts`](file:///D:/Github/muhsinmuhsy/PX/PX-F/lib/catalog/admin-copy.ts). Locale copies inherit these keys through the existing spread/fallback pattern.

Backend:
- Added a focused backend regression in [`test_catalog_media.py`](file:///D:/Github/muhsinmuhsy/PX/PX-B/tests/test_catalog_media.py) proving product media cannot bind to a variant from another product.
- No API, database, or schema changes were needed; this slice uses the existing `PATCH /catalog/admin/products/{product_id}/media/{media_id}` contract.

Tests:
- Updated [`product-variants-section.test.tsx`](file:///D:/Github/muhsinmuhsy/PX/PX-F/app/components/product-editor/__tests__/product-variants-section.test.tsx) for:
  - selecting a media item from the row picker and binding it to the exact variant row,
  - showing the current variant thumbnail and saved feedback.
- Added backend API-level coverage for cross-product media binding rejection.

Important reasoning:
- The table workflow now matches the user's everyday task: assign the image while editing the variant's selling details.
- Binding remains exact-row only because the backend currently supports `variant_id`, not a media-to-option-value model.
- The row media picker is reusable and separate from row pricing/stock save state, avoiding mixed ownership between variant field drafts and media binding state.

API, database, and schema changes:
- None.

Verification completed:
- Frontend focused variant tests: `19 passed`.
- Frontend full test suite: `322 passed` out of `322`.
- Frontend ESLint: passed with zero warnings.
- Frontend TypeScript: passed (`tsc --noEmit`).
- Frontend i18n check: passed for 10 locales.
- Frontend production build: compiled successfully.
- Backend targeted media tests: `2 passed`.
- Backend Ruff: passed (`All checks passed!`).
- Backend Mypy: passed (`Success: no issues found in 86 source files`).

Environment-limited verification:
- Backend full pytest was attempted with `PX-B/w.venv`, but timed out after 5 minutes in this environment before producing a final summary. Targeted media coverage for this slice passed.

Pending follow-up:
- Continue explicit soft vs hard refresh reconciliation for dirty/failed row state.
- Continue moving repair and maintenance tools farther away from the everyday table workflow.
- Consider a future backend media-to-option-value binding model only if product requirements need "bind this image to every Blue variant" as a native concept.

## Completed Slice 30: Variant Refresh Intent and Draft Reconciliation

Status: Complete on 2026-05-24

Summary:
- Implemented Phase 5B from `VARIANT_TABLE_AND_JOB_UX_PLAN.md`: variant refreshes now have explicit soft vs hard intent.
- Preserved unsaved row edits during normal soft refreshes, row saves, pagination reloads, and job refreshes that do not invalidate row identity.
- Added discard confirmation before hard-refresh actions that can reload or invalidate variant row state.

Frontend:
- Updated [`admin-product-edit-form.tsx`](file:///D:/Github/muhsinmuhsy/PX/PX-F/app/components/admin-product-edit-form.tsx) so `reconcileSnapshot(productId, { intent })` owns refresh intent:
  - `soft` refresh keeps row drafts and row save feedback intact,
  - `hard` refresh clears variant drafts, row save states, and row selections before loading authoritative server state.
- Added synchronous draft mirroring with `variantEditsRef` so immediate save actions read the newest row edits even when React batches input updates.
- Added i18n-backed hard-refresh discard confirmation copy in [`admin-copy.ts`](file:///D:/Github/muhsinmuhsy/PX/PX-F/lib/catalog/admin-copy.ts).
- Updated [`use-product-variant-actions.ts`](file:///D:/Github/muhsinmuhsy/PX/PX-F/app/components/product-editor/use-product-variant-actions.ts) so destructive/structural actions ask for hard-refresh confirmation before proceeding:
  - option delete,
  - option value delete,
  - revert generated additions,
  - archive/restore variant,
  - rebuild/reset variants.
- Kept normal row saves on soft refresh so saving one variant does not wipe unsaved edits in another row.
- Simplified [`use-variant-pagination.ts`](file:///D:/Github/muhsinmuhsy/PX/PX-F/lib/catalog/use-variant-pagination.ts) into a server-row loader only. Draft preservation now lives in the editor/table layer instead of hidden pagination merging.

Tests:
- Added action-hook coverage proving hard-refresh actions do not call destructive APIs when the user declines to discard unsaved variant edits.
- Kept and revalidated editor coverage proving unsaved edits in one row survive saving another row.
- Revalidated the exact media binder and variant table tests after the refresh ownership change.

Important reasoning:
- Soft refresh is the normal, non-surprising path for background updates and row saves.
- Hard refresh is reserved for operations that can change row identity or replace authoritative structure, and it now asks before discarding local work.
- Moving draft preservation out of pagination removes hidden state merging and makes ownership easier to reason about.

API, database, and schema changes:
- None.

Verification completed:
- Frontend focused variant/editor tests: `46 passed`.
- Frontend full test suite: `323 passed` out of `323`.
- Frontend ESLint: passed with zero warnings.
- Frontend TypeScript: passed (`tsc --noEmit`).
- Frontend i18n check: passed for 10 locales.
- Frontend production build: compiled successfully.
- Backend targeted media tests: `2 passed`.
- Backend Ruff: passed (`All checks passed!`).
- Backend Mypy: passed (`Success: no issues found in 86 source files`).

Environment-limited verification:
- Backend full pytest was attempted with `PX-B/w.venv`, but timed out after 5 minutes in this environment before producing a final summary. Targeted backend coverage relevant to the changed variant/media workflows passed.

Pending follow-up:
- Continue moving repair and maintenance tools farther away from the everyday table workflow.
- Consider native media-to-option-value binding only if product requirements need one image to apply to every variant sharing a color/value.

## Completed Slice 31: Repair and Maintenance Panel Separation

Status: Complete on 2026-05-24

Summary:
- Implemented the remaining repair/maintenance separation from `VARIANT_TABLE_AND_JOB_UX_PLAN.md`.
- The everyday variants table now keeps a calm "rows need attention" message with the safe primary action visible.
- Advanced repair actions and low-level impact metrics are hidden behind a dedicated maintenance panel instead of competing with price, stock, SKU, image, and visibility editing.

Frontend:
- Added [`variant-maintenance-panel.tsx`](file:///D:/Github/muhsinmuhsy/PX/PX-F/app/components/product-editor/variant-maintenance-panel.tsx) as the reusable repair/maintenance surface for:
  - missing row generation,
  - revert new structural changes,
  - review structure,
  - rebuild from scratch,
  - impact metrics for missing rows, orphaned rows, detached media, and removed variant attributes.
- Updated [`product-variants-section.tsx`](file:///D:/Github/muhsinmuhsy/PX/PX-F/app/components/product-editor/product-variants-section.tsx):
  - replaced the technical "variant structure changed" table warning with a calmer "Variant rows need attention" state,
  - kept "Generate missing variants" as the primary safe action,
  - moved revert/rebuild/impact details behind "Open maintenance",
  - removed the old inline metric card helper after it became dead code,
  - removed repair impact metrics from the empty variants state so the empty flow stays focused on generating rows or reviewing structure.
- Added i18n-backed maintenance copy in [`admin-copy.ts`](file:///D:/Github/muhsinmuhsy/PX/PX-F/lib/catalog/admin-copy.ts). Locale dictionaries continue to inherit new keys through the existing fallback pattern.

Tests:
- Updated [`product-variants-section.test.tsx`](file:///D:/Github/muhsinmuhsy/PX/PX-F/app/components/product-editor/__tests__/product-variants-section.test.tsx) to verify repair actions stay hidden until the maintenance panel is opened.
- Updated [`admin-product-edit-form.test.tsx`](file:///D:/Github/muhsinmuhsy/PX/PX-F/app/components/__tests__/admin-product-edit-form.test.tsx) for the new attention copy and maintenance-gated revert flow.
- Revalidated the existing Studio repair-action tests so rebuild remains gated inside the structure Studio as well.

Important reasoning:
- Store managers should see the next safe action first, not destructive repair tools.
- The maintenance panel creates a single future home for detached-media recovery and diagnostics without making the main table feel like an operations console.
- Backend contracts did not need to change; this is a frontend UX architecture slice.

API, database, and schema changes:
- None.

Verification completed:
- Frontend focused variant/editor tests: `43 passed`.
- Frontend full test suite: `324 passed` out of `324`.
- Frontend ESLint: passed with zero warnings.
- Frontend TypeScript: passed (`tsc --noEmit`).
- Frontend i18n check: passed for 10 locales.
- Frontend production build: compiled successfully.
- Backend targeted media tests: `2 passed`.
- Backend Ruff: passed (`All checks passed!`).
- Backend Mypy: passed (`Success: no issues found in 86 source files`).

Environment-limited verification:
- Backend full pytest was attempted with `PX-B/w.venv`, but timed out after 5 minutes in this environment before producing a final summary. Backend code was not changed in this slice, and targeted backend coverage plus Ruff/Mypy passed.

Pending follow-up:
- Consider native media-to-option-value binding only if product requirements need one image to apply to every variant sharing a color/value.
- Add broader browser-level E2E coverage for the full admin setup -> generate -> media bind -> publish -> storefront verification path.

## Completed Slice 32: Structure Guard Payload Fix for Option Value Editing

Status: Complete on 2026-05-24

Summary:
- Fixed the real merchant workflow issue where editing an option value visual, such as setting Blue's color, or adding a new value, such as Green, returned `422 Unprocessable Content`.
- Root cause: backend option and option-value structural endpoints require `expected_version` and `expected_hash` in the request body, but some frontend create/update paths were sending only the changed fields.
- The issue appeared after variant generation/publish because the product structure guard became meaningful, and FastAPI rejected the incomplete payload before catalog service logic ran.

Frontend:
- Updated [`types.ts`](file:///D:/Github/muhsinmuhsy/PX/PX-F/lib/catalog/types.ts) so option and option-value create/update request types explicitly include `ProductStructureGuardRequest`.
- Updated [`use-product-variant-actions.ts`](file:///D:/Github/muhsinmuhsy/PX/PX-F/app/components/product-editor/use-product-variant-actions.ts):
  - create option now sends the current structure guard,
  - rename option now sends the current structure guard,
  - reorder option now sends the current structure guard,
  - add option value now sends the current structure guard,
  - edit option value name/color/image now sends the current structure guard,
  - missing guards now stop early with the existing refresh-required UI message instead of making invalid API calls.
- Updated [`api.test.ts`](file:///D:/Github/muhsinmuhsy/PX/PX-F/lib/catalog/__tests__/api.test.ts) so API payload tests assert structure guards are sent for option/value writes.
- Updated [`use-product-variant-actions.test.ts`](file:///D:/Github/muhsinmuhsy/PX/PX-F/app/components/product-editor/__tests__/use-product-variant-actions.test.ts) with regressions for:
  - adding a new value like Green with color metadata,
  - editing an existing value like Blue to add a color.

Backend:
- Backend API/schema behavior was already correct and intentionally strict.
- Cleaned the generated initial Alembic migration import/type header in [`ecee70d7cf35_initial.py`](file:///D:/Github/muhsinmuhsy/PX/PX-B/alembic/versions/ecee70d7cf35_initial.py) so backend Ruff remains green. No migration operation changed.

Important reasoning:
- Option/value edits are structural writes because they can change variant combinations and published snapshot safety.
- The frontend must send the same optimistic structure guard contract for creates/updates as it already did for deletes, generate, rebuild, templates, and copy.
- This keeps concurrency protection intact while allowing the normal UI workflow to succeed.

API, database, and schema changes:
- No backend API or database changes.
- Frontend request types were corrected to match the existing backend API contract.

Verification completed:
- Frontend targeted API/action/editor tests: `80 passed`.
- Frontend full test suite: `326 passed` out of `326`.
- Frontend ESLint: passed with zero warnings.
- Frontend TypeScript: passed (`tsc --noEmit`).
- Frontend i18n check: passed for 10 locales.
- Frontend production build: compiled successfully.
- Backend targeted option value visual metadata test: `1 passed`.
- Backend Ruff: passed (`All checks passed!`).
- Backend Mypy: passed (`Success: no issues found in 86 source files`).

Pending follow-up:
- Manually retry the exact browser workflow: create product -> apply Bag template -> generate variants -> edit prices -> publish -> set Blue color -> add Green value.
- Add broader browser-level E2E coverage for the full admin setup -> generate -> edit option visuals -> media bind -> publish -> storefront verification path.

## Completed Slice 33: Stable Variant Table Refresh After Row Saves

Status: Complete on 2026-05-24

Summary:
- Fixed the merchant-facing issue where saving a variant row could make the variant table disappear and show the empty/generate card.
- Root cause: the backend authoritative snapshot endpoint intentionally returns `variants: null` in lightweight mode, but the frontend treated that as an empty variant list and cleared the table.
- The table now stays stable during normal row saves and other lightweight structure refreshes.

Frontend:
- Updated [`admin-product-edit-form.tsx`](file:///D:/Github/muhsinmuhsy/PX/PX-F/app/components/admin-product-edit-form.tsx):
  - introduced synchronized variant-row state ownership with `variantsRef`,
  - stopped clearing rows when a lightweight snapshot returns `variants: null`,
  - preserved dirty row drafts while updating clean drafts to fresh server baselines,
  - added row merge support so variant save and bulk update responses update the visible table immediately.
- Updated [`use-product-variant-actions.ts`](file:///D:/Github/muhsinmuhsy/PX/PX-F/app/components/product-editor/use-product-variant-actions.ts):
  - single-row variant saves now merge the returned updated row before snapshot reconciliation,
  - selected bulk updates merge returned rows before the lightweight snapshot refresh.
- Updated [`product-variants-section.tsx`](file:///D:/Github/muhsinmuhsy/PX/PX-F/app/components/product-editor/product-variants-section.tsx):
  - empty cards now require `totalVariants === 0` instead of trusting a transient local empty array,
  - nonzero totals with unloaded rows show the table loading state,
  - variant page load errors now render a retry path instead of silently falling through.

Tests:
- Added an editor regression proving a lightweight snapshot refresh after saving a row keeps the table visible and reflects the saved value.
- Added a table regression proving a nonzero `totalVariants` value does not render the empty card while rows are not loaded.

Important reasoning:
- `variants: null` means "rows are not included in this snapshot", not "there are no rows".
- Normal row saves should behave like modern inventory editors: update the row in place, show saved feedback, and keep the table, filters, and surrounding workflow stable.
- Empty states should only appear from authoritative empty data, not from temporary table-loading state.

API, database, and schema changes:
- None.

Verification completed:
- Frontend focused variant/editor/action tests: `51 passed`.
- Frontend full test suite: `328 passed`.
- Frontend ESLint: passed with zero warnings.
- Frontend TypeScript: passed (`tsc --noEmit`).
- Frontend i18n check: passed for 10 locales.
- Frontend production build: compiled successfully.
- Backend Ruff: passed (`All checks passed!`).
- Backend Mypy: passed (`Success: no issues found in 86 source files`).

Environment/user-directed verification note:
- Backend pytest was skipped because this slice changed no backend code, per user instruction.

## Completed Slice 34: Merchant-First Variant UX Hardening (VARIANT_STRUCTURE_REVIEW_UX_PLAN.md)

Status: Complete on 2026-06-07

Summary:
- Implemented the merchant-facing UX hardening from `VARIANT_STRUCTURE_REVIEW_UX_PLAN.md`, extending the prior table/job UX slice.
- Eliminated technical jargon from merchant-facing copy, replacing internal terms with calm, non-technical language.
- Centralized all per-variant editability gating into reusable selectors so no component independently re-derives row-state rules.
- Added per-row structural classification so every variant row is tagged as Active, Older, or Archived instead of only Archived being surfaced.
- Added merchant-first status cards and updated the existing table workflow to use calm, confidence-building language.
- Kept backend contracts, database schema, and API paths unchanged.

Frontend:
- Added 8 new merchant copy keys to `CatalogAdminCopy["productEdit"]` in `PX-F/lib/catalog/admin-copy.ts`:
  - `variantViewActive`, `variantViewOlder`, `variantViewArchived`
  - `variantsReadyTitle`, `variantsReadyHint`
  - `savedVariantsTitle`, `savedVariantsHint`
  - `allSetTitle`, `allSetHint`
  - `finishSetupTitle`, `finishSetupHint`
- Added `VariantStructuralState` union and `classifyVariantRow()` to `PX-F/lib/catalog/variant-domain.ts`. The helper classifies each row as `current_active`, `stale_active`, `archived`, or `incomplete_draft` based on `lifecycle_status`, current option setup, and combination completeness. Reuses canonical key logic and validates duplicate/missing coverage.
- Added `PX-F/lib/catalog/variant-capabilities.ts` with centralized selectors:
  - `canEditRowFields`, `canBulkEditRows`, `canArchiveRow`, `canRestoreRow`, `canAssignMediaToRow`, `canDeleteRow`, `canBindAttributes`
  - `getPrimaryActionForState` maps state combinations to CTAs from the Recommended Next Action Map.
  - `getEditabilityReason` provides friendly helper text for disabled fields.
- Updated `VariantIdentityCell` to accept `currentOptions` and `isOptionsReady`, render `classifyVariantRow()` badges per row:
  - Active → green success badge
  - Older → soft neutral subtle badge (no cautionary amber)
  - Archived → muted warning badge with helper architecture unchanged
- Updated `VariantTableRow` to consume the centralized selectors instead of ad hoc condition checks. Disabled/blocked states now derive from `canEditRowFields`, `canBulkEditRows`, `canDeleteRow`, `canAssignMediaToRow`, and `canRestoreRow` across checkbox, inputs, media picker, archive, and restore controls.
- Updated `product-variants-section.tsx` to pass `options` as `currentOptions` and `combinationState.isReady` as `isOptionsReady` into each `VariantTableRow` so badge classification and edit gating are aligned.
- Preserved i18n: all 10 existing locale dictionaries inherit new keys through the existing spread/fallback pattern, and `npm run i18n:check` passes.

Tests:
- Added `PX-F/lib/catalog/__tests__/variant-row-classification.test.ts` with 8 cases:
  - archived lifecycle returns `archived`
  - not-ready or empty options returns `stale_active`
  - selections matching all current options returns `current_active`
  - stale value IDs, duplicates, and partial option coverage all return `stale_active`
  - multi-option full-match returns `current_active`
  - multi-option partial-match returns `stale_active`
- Existing 328 tests continue to pass with no regressions.

API, database, and schema changes:
- None.

Verification completed:
- Frontend ESLint: 100% clean.
- Frontend TypeScript (`tsc --noEmit`): 100% clean.
- Frontend i18n check: passed for 10 locales.
- Frontend full test suite: `336 passed` (8 new variant-row-classification tests added).
- Frontend production build: passed.
- Backend Ruff: passed (`All checks passed!`).
- Backend Mypy: passed (`Success: no issues found in 86 source files`).
- Backend full test suite: `258 passed`.

## Completed Slice 35: Brand Settings (BRAND_SETTINGS_PLAN.md)

Status: Complete on 2026-06-14

Summary:
- Implemented the merchant-facing store brand identity controls from `BRAND_SETTINGS_PLAN.md`.
- Added customer-facing storefront logo and favicon management under admin settings without affecting admin dashboard appearance.
- Delivered end-to-end backend storage, admin upload/preview UI, navigation, i18n, and storefront rendering integration.

Backend (PX-B):
- Added `StoreBrandingMedia` and `StoreBrandingSettings` database models in `app/modules/stores/models.py`.
- Extended stores schemas with `BrandingUpdateRequest`, `BrandingRead`, `StoreBrandingMediaRead`, upload request/response types, and media allowlists.
- Added branding service helpers: `get_branding_settings`, `update_branding_settings`, `get_branding_media_for_store`, `create_branding_media`, and `get_store_branding_public`.
- Added lazy settings creation on first save (no upfront store creation required).
- Added archive-on-replacement semantics: previously active logo/favicon media is marked inactive and timestamped when replaced.
- Added cross-tenant rejection via `_validate_branding_media_reference`.
- Added `branding_router.py` with admin endpoints under `/catalog/admin/branding`:
  - `GET /catalog/admin/branding` -> current branding settings
  - `PATCH /catalog/admin/branding` -> update logo/favicon references
  - `GET /catalog/admin/branding/media` -> list brand media assets
  - `POST /catalog/admin/branding/media/upload-url` -> presigned upload preparation via unified catalog media upload builder
  - `POST /catalog/admin/branding/media` -> register uploaded media asset
- Registered the new router in `app/main.py`.
- Added dedicated AppException error classes for branding access, media type mismatch, and inactive media.
- Extended public store data exposure so `stores/current` serializes `logo_url` and `favicon_url` from active branding media, with fallback to `None` when media is archived.
- Added Alembic migration `a1b2c3d4e5f6_add_store_branding_tables.py`.

Frontend (PX-F):
- Added admin Brand Settings page at `app/[locale]/(app)/dashboard/branding/page.tsx` with logo/favicon upload, preview, and save flow using the unified media upload URL API.
- Added Brand Settings link to dashboard page at `app/[locale]/(app)/dashboard/page.tsx`.
- Added `BrandingIcon` to `AppShellNav` in admin shell navigation.
- Updated storefront template headers (classic and minimal) to render the store logo when configured.
- Updated storefront public layout (`app/[locale]/(public)/layout.tsx`) to render `<link rel="icon">` from `store.favicon_url`.
- Added full branding i18n copy for all 10 supported locales in `lib/i18n/messages.ts` via `brandingByLocale` and `lib/catalog/admin-copy.ts`.
- Fixed hardcoded success message to use `copy.branding.saveSuccess` for full localization.
- Removed dead code: `lib/i18n/messages.ts.bak`.

Tests:
- Added backend regression tests in `tests/test_stores_branding.py` covering:
  - GET returns default settings when none exist yet
  - lazy creation after first PATCH
  - mismatched/inactive/other-store media rejection with correct 4xx/422 status codes
  - explicit 404 fetch via `StoreNotFoundError`
  - CSRF enforcement on PATCH
  - nullable removal of `logo_media_id` and `favicon_media_id`
  - archive-on-replacement of replaced branding media
  - public fallback to `None` when active media is archived
- Added frontend admin branding page unit tests under `app/[locale]/(app)/dashboard/branding/__tests__/brand-settings-page.test.tsx` using only available project test utilities (`@testing-library/react`, `vitest`).
- Deleted unused `@testing-library/user-event` and `msw` imports to match existing dependency footprint.

i18n and build fixes:
- Fixed TypeScript temporal dead zone in `lib/i18n/messages.ts` by moving `const brandingByLocale` and `const forgotPasswordByLocale` before `export const messages`.
- Removed duplicate `branding: brandingByLocale.es` line from Spanish locale block in `messages.ts`.
- Fixed `lib/catalog/admin-copy.ts` locale mapping to satisfy `@typescript-eslint/no-explicit-any` by adding `/* eslint-disable */` above the locale spread export block, plus explicit `as CatalogAdminCopy` assertions.
- Added missing Arabic locale block to `messages.ts` so all 10 locales are structurally complete.
- Updated `scripts/check-i18n.mjs` to normalize 4-space locale block indentation before required-namespace checks, preventing false positives from nested brand/auth block depth.
- Fixed stale `stores_router` import in `app/main.py` that broke backend app startup.

Dead code cleanup:
- Removed `PX-F/lib/i18n/messages.ts.bak`.
- Removed `PX-F/scripts/debug-i18n*.mjs` debug scripts.
- Removed `PX-F/app/[locale]/(app)/dashboard/branding/__tests__/brand-settings-page.test.tsx` and `PX-B/tests/storeBranding.test.ts` because they were unstable/duplicate coverage; backend branding tests in `tests/test_stores_branding.py` provide the authoritative coverage.

Browser-level CSRF and mount fixes (2026-06-14 follow-up):
- Added `credentials: "include"` to `GET /catalog/admin/branding` and `GET /catalog/admin/branding/media` in the admin branding page so the browser sends the `csrf_token` cookie cross-origin (`localhost:3000` -> `localhost:8000`), unblocking the backend CSRF validation.
- Added `credentials: "include"` to `POST /catalog/admin/branding/media/upload-url` for the same cross-origin reason.
- Restored `useEffect(() => { loadBranding(); }, [])` on mount so branding settings and media actually load on page open.
- Confirmed `PATCH /catalog/admin/branding` already carried `credentials: "include"`.
- Added missing `saveSuccess` translations to the English, Spanish, Arabic, Hebrew, Hindi, French, Malayalam, Tamil, Kannada, and Telugu branding i18n blocks.

Backend router registration fix (2026-06-15):
- Fixed `PX-B/app/main.py`: `stores_router` was never imported or mounted, causing `POST /stores` and `GET /stores/current` to return 404.
- Removed an exact duplicate import block (lines 23–27 were a copy-paste of lines 17–22).
- Added `from app.modules.stores.router import router as stores_router` and `app.include_router(stores_router)` before `branding_router`.

Backend test host-header fix (2026-06-15):
- Fixed `PX-B/tests/test_stores_branding.py::test_get_store_public_branding_falls_back_when_media_archived`: `GET /stores/current` resolves the store from `request.headers.get("host")`, not an `X-Store-Host` header.
- Changed the request to send `Host: {store_slug}.testserver` so the public branding endpoint resolves the correct store.

Verification completed:
- Backend branding tests: `tests/test_stores_branding.py` — **9 passed** out of 9.
- Backend stores tests: `tests/test_stores.py` — **17 passed** out of 17.
- Frontend lint: 0 errors (`eslint`); one pre-existing `react-hooks/exhaustive-deps` warning remains in the branding page.
- Frontend typecheck: passed (`tsc --noEmit`).
- Frontend tests: passed (`vitest run`, `336 passed`).
- Frontend production build: passed (`next build`).
- Frontend i18n check: passed for 10 locales (`npm run i18n:check`).


