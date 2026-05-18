# Variants Stabilization Progress

Last updated: 2026-05-18

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
