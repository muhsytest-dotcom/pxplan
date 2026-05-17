# Agent Handoff and Continuity

Last updated: 2026-05-17

Read this first, then `00_START_HERE.md`, `WHAT-DONE.md`, and the architecture docs. Some older guide files still describe the pre-implementation state, so treat this file plus `WHAT-DONE.md` and the live code as the current source of truth.

## Latest Implementation Status

The variants stabilization work is largely complete through Slice 15.

Completed:
- Core variant domain foundation and canonical identity.
- Durable SSE job events with replay.
- Variant explosion protection and Basic/Pro variant limits.
- Decoupled worker boundary, error classification, retry tracking, and timeout/cancellation behavior.
- Immutable product structure snapshots and publication swaps.
- Backend i18n error-key envelopes and frontend catalog error translation.
- Variant job metrics, Prometheus-compatible instrumentation, and dashboard panel.
- Variant job SSE recovery fallback with visible live/polling state.
- Batched variant selection resolution for admin/storefront listing.
- Policy-based target-store tier enforcement for option-value growth, template application, and structure copy.
- API-level regression coverage for published snapshot storefront visibility while draft structure changes continue.
- API-level regression coverage for template quota failures and rebuild media rebinding.

Current quality gate from the latest slice:
- Backend full tests: `python -m pytest tests -q` passed.
- Backend lint: `.\.venv\bin\ruff check .` passed.
- Backend typecheck: `python -m mypy app` passed.
- Frontend full tests: `npm test` passed (`301 passed`).
- Frontend lint: `npm run lint` passed.
- Frontend typecheck: `npm run typecheck` passed.
- Frontend i18n check: `npm run i18n:check` passed for 10 locales.
- Frontend production build: `npm run build` passed.

## Summary of Latest Completed Work

Slice 15 added release-hardening API regressions for two remaining critical workflows:
- Added `test_template_apply_api_returns_i18n_quota_error_for_basic_store` in `PX-B/tests/test_catalog_tier_policy.py`.
- Verified oversized store-scoped template application returns `PRODUCT_VARIANT_TIER_QUOTA_EXCEEDED` with `catalog.variant.errors.tier_quota_exceeded` and `{ tier, limit, projected }` context.
- Added `test_rebuild_variants_marks_detached_media_for_rebinding` in `PX-B/tests/test_catalog_variants.py`.
- Verified rebuild impact reports media detachment and rebuild marks detached media with `needs_variant_rebinding = true`.
- No runtime behavior, API path, database, or schema changes were made.

## Current Architecture Decisions

- **Published snapshot owns storefront structure**: Once `product.published_version` is set, storefront structure reads must resolve from the matching immutable `CatalogProductStructureSnapshot`, not live draft options.
- **Published version owns storefront variants**: Storefront variant listing must remain scoped to variants generated for `product.published_version` and must not expose newer draft structures before publication.
- **Target-store policy wins**: Template application and structure copy must enforce the destination product/store tier, not the source/template origin.
- **Quota errors stay translatable at API boundaries**: Template quota failures must preserve the `AppException` envelope with `error.i18n_key` and interpolation `context`.
- **Rebuild media rebinding remains visible**: Full rebuilds that remove old variants must detach affected media and mark it for rebinding so admins can reconnect media to generated variants.
- **Block invalid structures early**: Public structure writes should refuse projected over-quota matrices before the user reaches rebuild/job execution.
- **Keep legacy safeguards**: Rebuild preview and job creation must still detect over-quota data that may come from migrations, imports, or older records.
- **Error envelopes stay i18n-compatible**: New user-facing policy failures should include `error.i18n_key` and `error.context`.

## Important Reasoning Behind Changes

Slice 15 was test-focused because the underlying policy and rebinding behavior already existed, but the API contracts needed explicit coverage. It intentionally avoided archive-vs-delete implementation because that remains a higher-risk product/domain decision involving historical references and database constraints.

## Pending Tasks

Still pending from the broader plan:
- Final archive-vs-delete migration strategy for variants and historical references.
- Broader E2E coverage for critical admin and storefront flows. Published snapshot visibility, template quota failure, and media rebinding after rebuild now have API-level regression coverage; full browser-level flows still need coverage.
- Final dead-code cleanup after remaining design decisions settle.
- Optional deeper observability: alert policy surfaces, health views, reconnect metrics, and operational drilldowns.
- Keep planning docs aligned when future slices land.

## Known Issues or Blockers

- Some historical notes in `WHAT-DONE.md` describe earlier blocker work as it existed at that time. Treat those as history, not current instructions.
- Archive-vs-delete is still a real product/domain decision. Avoid hard-deleting historical variants further until that migration strategy is finalized.

## Important Files and Modules

Backend:
- `PX-B/app/modules/catalog/service.py`: Core variant business logic, snapshots, jobs, quota enforcement, selection resolution.
- `PX-B/app/modules/catalog/router.py`: Catalog admin/storefront API routes.
- `PX-B/app/modules/catalog/variant_domain.py`: Shared domain constants and canonical identity helpers.
- `PX-B/app/modules/catalog/variant_events.py`: Durable SSE event bus.
- `PX-B/app/modules/catalog/worker.py`: Decoupled job worker and retry execution.
- `PX-B/app/exceptions/errors.py`: App exception classes and i18n error metadata.
- `PX-B/app/core/i18n_keys.py`: Backend i18n key registry.
- `PX-B/app/core/rate_limit/policies.py`: Rate-limit policy registry.

Frontend:
- `PX-F/lib/catalog/api.ts`: Catalog API client.
- `PX-F/lib/catalog/types.ts`: Frontend contract types.
- `PX-F/lib/catalog/i18n.ts`: Catalog error-key translation.
- `PX-F/lib/catalog/variant-job-events.ts`: Variant job SSE helpers.
- `PX-F/app/components/admin-product-edit-form.tsx`: Product edit orchestration.
- `PX-F/app/components/product-editor/product-variants-section.tsx`: Variant management UI and job panel.
- `PX-F/app/components/variant-job-metrics-panel.tsx`: Operational metrics dashboard panel.

Tests:
- `PX-B/tests/test_catalog_variants.py`
- `PX-B/tests/test_catalog_tier_policy.py`
- `PX-B/tests/test_catalog_variant_ops.py`
- `PX-B/tests/test_catalog_variant_observability.py`
- `PX-B/tests/test_catalog_variant_query_efficiency.py`
- `PX-F/app/components/__tests__/variant-job-metrics-panel.test.tsx`
- `PX-F/app/components/product-editor/__tests__/product-variants-section.test.tsx`

## API, Database, and Schema Changes

Latest slice:
- No database migration.
- No public API path changes.
- No schema changes.
- Test-only coverage added for existing template quota API failures and rebuild media rebinding.

Earlier completed work added job error tracking fields, durable variant job events, snapshots, metrics response types, and rate-limit policies. See `WHAT-DONE.md` for slice-by-slice details.

## Current Implementation Direction

The project should now move from stabilization into release hardening:
- Decide and implement archive-vs-delete migration.
- Add the remaining E2E/API regression tests around the most valuable flows.
- Clean stale docs and dead code.
- Keep quality gates full and boring: backend tests/lint/mypy; frontend tests/lint/typecheck/i18n/build.

## Assumptions Made

- Unknown or missing store tier is treated as Basic.
- `pro` is the only higher tier currently recognized by `get_variant_limit_for_tier`.
- Public structure mutations should be prevented from creating over-quota matrices, even if generation is not immediately requested.
- Legacy/imported over-quota structures may still exist and must remain safely blocked at preview/job time.
- Snapshot publication remains explicit through `/catalog/admin/products/{product_id}/snapshots/{snapshot_id}/publish`; generation jobs do not auto-publish.

## Recommended Next Steps

1. Keep plan/checklist docs current when Slices 15+ land.
2. Plan the archive-vs-delete migration carefully before touching variant deletion semantics.
3. Add remaining browser-level E2E coverage for full admin setup/generate/publish/storefront visibility.
4. Run the full quality gate after every implementation slice.

## Things Future Agents Should Avoid Breaking

- Do not bypass `assert_no_active_job` for structure mutations.
- Do not remove rebuild/job quota checks; they protect legacy/imported data.
- Do not hard-delete variants that may have historical references without the archive migration plan.
- Do not make storefront structure read live draft options when a published snapshot exists.
- Do not make storefront variants ignore `product.published_version`.
- Do not strip quota failure `error.i18n_key` or `error.context` metadata.
- Do not stop marking detached media as needing rebinding after rebuild/template replacement flows.
- Do not change error envelope shape; frontend error/i18n handling depends on it.
- Do not reintroduce per-variant selection N+1 loading in list endpoints.
- Do not make SSE execution perform work; SSE streams status only.
- Do not update `WHAT-DONE.md` until implementation and validation for a slice are complete.
