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

## Completed Slice 8: Blocker Cleanup and Full Quality Gate Refresh

Status: Complete on 2026-05-16

This pass re-read the implementation guides, verified that the three BLOCKER tracks were present in the codebase, and tightened the remaining rough edges around i18n envelopes, translation coverage, lint warnings, and validation.

Backend:
- Added reusable catalog HTTP error detail helpers in `PX-B/app/modules/catalog/router.py` so catalog routes can consistently return `message`, `i18n_key`, and optional interpolation `context`.
- Updated variant job create/rebuild/active/detail/cancel/SSE route not-found responses to include catalog i18n keys.
- Updated the product structure guard HTTP conflict response to include `catalog.variant.errors.structure_stale_hash` and context.
- Added a backend regression test asserting that variant job creation for a missing product returns `error.i18n_key = catalog.product.errors.not_found`.

Frontend:
- Completed French and German catalog variant error translations in `PX-F/px/lib/catalog/i18n.ts`.
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

## Still Not Complete

The full multi-phase plan is not finished yet. Remaining major areas:
- **Phase 9: Observability & Metrics**: Deeper health views and alert policy surfaces can be added later, but core backend metrics and dashboard visibility are now complete.
- **Policy-Based Tier Enforcement**: Variant structure write enforcement is now complete; any future non-variant catalog tier features should follow the same target-store policy pattern.
- **Archive-vs-delete migration**: Finalizing the strategy for existing variants and historical references.
- **E2E coverage**: Broader E2E coverage for critical admin and storefront workflows.
- **Dead-code cleanup**: Final pass across both apps after all phases are complete.
