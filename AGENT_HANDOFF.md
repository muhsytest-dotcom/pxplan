# Agent Handoff and Continuity

Last updated: 2026-05-18

Read this first, then `00_START_HERE.md`, `WHAT-DONE.md`, and the architecture docs. Some older guide files still describe earlier slice numbers, so treat this file plus `WHAT-DONE.md` and the live code as the current source of truth.

## Latest Implementation Status

The variants stabilization work is complete through Slice 18.

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
- Frontend admin workflow for explicitly publishing generated product structure snapshots to storefront.
- PostgreSQL backend test isolation and FK-ordering hardening, with the full Postgres suite passing.
- Product variant lifecycle persistence and direct archive-on-remove behavior, with active-only variant uniqueness.

Current quality gate from the latest slice:
- Backend PostgreSQL full suite passed: `PATH="$PWD/.venv/bin:$PATH" python -m dotenv -f .env.local.pg run -- pytest -v` (`248 passed`, `209 warnings`).
- Backend Ruff passed: `.venv/bin/ruff check .`.
- Backend mypy passed: `.venv/bin/python -m mypy app`.
- Frontend full tests passed: `npm test` (`303 passed`).
- Frontend lint passed: `npm run lint`.
- Frontend typecheck passed: `npm run typecheck`.
- Frontend i18n check passed: `npm run i18n:check` for 10 locales.
- Frontend production build passed: `npm run build`.

## Summary of Latest Completed Work

Slice 18 implemented the first archive-vs-delete migration slice:
- Added persisted `ProductVariant.lifecycle_status` with `active`, `archived`, and `deleted` lifecycle values.
- Added Alembic migration `d4f2a9b7c801_add_product_variant_lifecycle_status.py`.
- Replaced all-row product variant SKU/combination uniqueness with active-only unique indexes so archived rows can remain historical without blocking active replacements.
- Updated direct admin variant removal to archive rows, mark them inactive, hide them from normal listings, and keep media rebinding behavior.
- Kept option/value deletion FK-clean by removing value links when the option/value row itself is deleted.
- Added `lifecycle_status` to backend/frontend variant read contracts and adjusted default admin copy to describe archiving.
- Added regression coverage proving archived variants are retained, hidden from normal discovery, and can be replaced by a new active row.

## Current Architecture Decisions

- **Published snapshot owns storefront structure**: Once `product.published_version` is set, storefront structure reads must resolve from the matching immutable `CatalogProductStructureSnapshot`, not live draft options.
- **Published version owns storefront variants**: Storefront variant listing must remain scoped to variants generated for `product.published_version` and must not expose newer draft structures before publication.
- **Publication remains explicit**: Generation jobs create snapshots but do not publish them. Admins publish the chosen snapshot through the explicit publish endpoint.
- **Frontend publish visibility is version-driven**: The publish panel appears only when the latest generated snapshot version is newer than the product's published version.
- **Target-store policy wins**: Template application and structure copy must enforce the destination product/store tier, not the source/template origin.
- **Variant removal archives by default**: Direct admin variant removal should set `lifecycle_status = archived` and `is_active = false`, not hard-delete the row.
- **Active-only uniqueness owns current catalog identity**: Variant SKU and combination uniqueness applies to active lifecycle rows so archived historical rows do not block replacement variants.
- **Quota errors stay translatable at API boundaries**: Template quota failures must preserve the `AppException` envelope with `error.i18n_key` and interpolation `context`.
- **Rebuild media rebinding remains visible**: Full rebuilds that remove old variants must detach affected media and mark it for rebinding so admins can reconnect media to generated variants.
- **Block invalid structures early**: Public structure writes should refuse projected over-quota matrices before the user reaches rebuild/job execution.
- **Keep legacy safeguards**: Rebuild preview and job creation must still detect over-quota data that may come from migrations, imports, or older records.
- **Error envelopes stay i18n-compatible**: New user-facing policy failures should include `error.i18n_key` and `error.context`.

## Important Reasoning Behind Changes

Slice 18 focused on the unresolved archive-vs-delete direction without overreaching into archive browsing or restore UX. Persisting lifecycle state and active-only uniqueness gives the domain a safe foundation: archived variants are retained for future history/reference views while normal admin and storefront discovery continue to show only active lifecycle rows.

The implementation intentionally keeps option/value deletion FK-clean. When an option value is deleted, affected variants are archived, but value links to that deleted row are removed before the value row is removed. Direct variant removal, where the option/value rows remain, preserves value links.

## Pending Tasks

Still pending from the broader plan:
- Richer archive/history behavior if product requirements need archive browsing, restore, or order/cart reference views.
- Broader browser-level E2E coverage for full admin setup -> generate -> publish -> storefront visibility.
- Final dead-code cleanup after remaining design decisions settle.
- Optional deeper observability: alert policy surfaces, health views, reconnect metrics, and operational drilldowns.
- Keep planning docs aligned when future slices land.

## Known Issues or Blockers

- The backend venv in this workspace is Linux-style. Ensure `.venv/bin` is on `PATH` when using the exact dotenv command shape, because `python -m dotenv -f .env.local.pg run -- pytest -v` otherwise may not find `pytest` in a non-activated shell.
- Some historical notes in `WHAT-DONE.md` describe earlier blocker work as it existed at that time. Treat those as history, not current instructions.
- Archive-vs-delete has an initial implementation for direct variant removal. Avoid expanding hard-delete behavior or adding restore/archive browsing without preserving active-only uniqueness and FK-clean option/value deletion semantics.

## Important Files and Modules

Backend:
- `PX-B/app/modules/catalog/service.py`: Core variant business logic, snapshots, jobs, quota enforcement, selection resolution, and FK-sensitive delete ordering.
- `PX-B/app/modules/catalog/models.py`: Product variant lifecycle status and active-only uniqueness indexes.
- `PX-B/app/modules/catalog/repository.py`: Variant listing/count/occupancy helpers default to active lifecycle rows.
- `PX-B/app/modules/catalog/router.py`: Catalog admin/storefront API routes; Slice 16 only changed the metrics route query annotation for Ruff compliance.
- `PX-B/app/modules/catalog/variant_domain.py`: Shared domain constants and canonical identity helpers.
- `PX-B/app/modules/catalog/variant_events.py`: Durable SSE event bus.
- `PX-B/app/modules/catalog/worker.py`: Decoupled job worker and retry execution.
- `PX-B/app/exceptions/errors.py`: App exception classes and i18n error metadata.
- `PX-B/app/core/i18n_keys.py`: Backend i18n key registry.
- `PX-B/app/core/rate_limit/policies.py`: Rate-limit policy registry.
- `PX-B/tests/conftest.py`: SQLite/PostgreSQL test harness and schema isolation.
- `PX-B/alembic/env.py`: Alembic migration environment and logging setup.
- `PX-B/alembic/versions/d4f2a9b7c801_add_product_variant_lifecycle_status.py`: Product variant lifecycle migration.

Frontend:
- `PX-F/lib/catalog/api.ts`: Catalog API client, now includes structure snapshot list/publish methods.
- `PX-F/lib/catalog/types.ts`: Frontend contract types, now includes snapshot/published version fields, `CatalogProductStructureSnapshotRead`, and optional variant `lifecycle_status`.
- `PX-F/lib/catalog/admin-copy.ts`: Catalog admin UI copy, including snapshot publish keys and archive-oriented default variant removal copy.
- `PX-F/lib/catalog/i18n.ts`: Catalog error-key translation.
- `PX-F/lib/catalog/variant-job-events.ts`: Variant job SSE helpers.
- `PX-F/app/components/admin-product-edit-form.tsx`: Product edit orchestration and snapshot publish wiring.
- `PX-F/app/components/product-editor/product-variants-section.tsx`: Variant management UI, job panel, and snapshot publish panel.
- `PX-F/app/components/variant-job-metrics-panel.tsx`: Operational metrics dashboard panel.

Tests:
- `PX-B/tests/test_catalog_variants.py`
- `PX-B/tests/test_catalog_tier_policy.py`
- `PX-B/tests/test_catalog_variant_ops.py`
- `PX-B/tests/test_catalog_variant_observability.py`
- `PX-B/tests/test_catalog_variant_query_efficiency.py`
- `PX-B/tests/test_store_domains.py`
- `PX-F/lib/catalog/__tests__/api.test.ts`
- `PX-F/app/components/__tests__/admin-product-edit-form.test.tsx`
- `PX-F/app/components/product-editor/__tests__/product-variants-section.test.tsx`
- `PX-F/app/components/__tests__/variant-job-metrics-panel.test.tsx`

## API, Database, and Schema Changes

Latest slice:
- No API path changes.
- Added `product_variants.lifecycle_status`, defaulting existing rows to `active`.
- Added active-only product variant unique indexes:
  - `uq_product_variants_active_product_combination_version`
  - `uq_product_variants_active_product_sku_version`
- Dropped the previous all-row `product_id + combination_key + version` and `product_id + sku + version` unique constraints.
- `ProductVariantRead` now includes `lifecycle_status`.

Previous Slice 16:
- Frontend API client calls existing backend endpoints:
  - `GET /catalog/admin/products/{product_id}/snapshots`
  - `POST /catalog/admin/products/{product_id}/snapshots/{snapshot_id}/publish`
- Frontend type contracts expose `ProductRead.snapshot_version`, `ProductRead.published_version`, `CatalogVariantJobRead.snapshot_id`, and `CatalogProductStructureSnapshotRead`.
- No backend API path changes.
- No database migration.
- No backend schema changes.

Earlier completed work added job error tracking fields, durable variant job events, snapshots, metrics response types, and rate-limit policies. See `WHAT-DONE.md` for slice-by-slice details.

## Current Implementation Direction

The project should continue release hardening:
- Add browser-level E2E coverage for admin setup -> generate -> publish -> storefront visibility.
- Extend archive/history behavior only where product requirements need explicit archive browsing, restore, or historical reference views.
- Clean stale/dead code after archive/history semantics settle.
- Keep quality gates explicit, and run backend gates against PostgreSQL for backend changes.

## Assumptions Made

- Unknown or missing store tier is treated as Basic.
- `pro` is the only higher tier currently recognized by `get_variant_limit_for_tier`.
- Public structure mutations should be prevented from creating over-quota matrices, even if generation is not immediately requested.
- Legacy/imported over-quota structures may still exist and must remain safely blocked at preview/job time.
- Snapshot publication remains explicit through `/catalog/admin/products/{product_id}/snapshots/{snapshot_id}/publish`; generation jobs do not auto-publish.
- The admin publish panel should be hidden unless a generated snapshot version is newer than the currently published storefront version.
- Direct variant removal archives the variant and hides it from default discovery; option/value deletion archives affected variants but removes value links to deleted rows for FK safety.
- Product variant SKU/combination uniqueness is active-only; archived rows must not block active replacement rows.

## Recommended Next Steps

1. Add browser-level E2E coverage for the complete admin generate/publish/storefront flow.
2. Add explicit archive/history admin or restore workflows only after product requirements define how archived variants should be surfaced.
3. Continue final dead-code cleanup only after archive/history semantics are settled.
4. Keep plan/checklist docs current when Slices 16+ land.
5. Run full frontend gates and PostgreSQL backend gates after every implementation slice.

## Things Future Agents Should Avoid Breaking

- Do not bypass `assert_no_active_job` for structure mutations.
- Do not remove rebuild/job quota checks; they protect legacy/imported data.
- Do not hard-delete variants that may have historical references; direct variant removal archives them.
- Do not remove active-only variant uniqueness; archived rows must not block replacement active variants.
- Do not preserve variant value links when deleting the option/value row itself; that path must remain FK-clean.
- Do not make storefront structure read live draft options when a published snapshot exists.
- Do not make storefront variants ignore `product.published_version`.
- Do not auto-publish generated snapshots.
- Do not show the frontend publish action when `latestSnapshot.version <= product.published_version`.
- Do not strip quota failure `error.i18n_key` or `error.context` metadata.
- Do not stop marking detached media as needing rebinding after rebuild/template replacement flows.
- Do not change error envelope shape; frontend error/i18n handling depends on it.
- Do not reintroduce per-variant selection N+1 loading in list endpoints.
- Do not make SSE execution perform work; SSE streams status only.
- Do not update `WHAT-DONE.md` until implementation and validation for a slice are complete.
