# Agent Handoff and Continuity

Last updated: 2026-05-17

Read this first, then `00_START_HERE.md`, `WHAT-DONE.md`, and the architecture docs. Some older guide files still describe earlier slice numbers, so treat this file plus `WHAT-DONE.md` and the live code as the current source of truth.

## Latest Implementation Status

The variants stabilization work is complete through Slice 16.

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

Current quality gate from the latest slice:
- Frontend focused tests passed: `npm test -- lib/catalog/__tests__/api.test.ts app/components/product-editor/__tests__/product-variants-section.test.tsx app/components/__tests__/admin-product-edit-form.test.tsx` (`61 passed`).
- Frontend full tests passed: `npm test` (`303 passed`).
- Frontend lint passed: `npm run lint`.
- Frontend typecheck passed: `npm run typecheck`.
- Frontend i18n check passed: `npm run i18n:check` for 10 locales.
- Frontend production build passed: `npm run build`.
- Backend Ruff passed under WSL venv: `wsl bash -lc "cd /mnt/d/Github/muhsinmuhsy/PX/PX-B && .venv/bin/ruff check ."`.
- Backend mypy passed under WSL venv: `wsl bash -lc "cd /mnt/d/Github/muhsinmuhsy/PX/PX-B && .venv/bin/python -m mypy app"`.
- Backend store-domain module passed: `wsl bash -lc "cd /mnt/d/Github/muhsinmuhsy/PX/PX-B && .venv/bin/python -m pytest tests/test_store_domains.py -q"`.
- Backend full-suite note: one WSL full-suite run reached a single unrelated failure in `tests/test_store_domains.py::test_invalid_domain_returns_field_error` (`401` instead of expected `400`), while the isolated test and module both passed. A repeated full-suite run timed out before completion. This appears to be existing cross-test/order sensitivity, not caused by Slice 16 runtime logic.

## Summary of Latest Completed Work

Slice 16 wired frontend snapshot publication into the admin product variants workflow:
- Added frontend contract support for structure snapshots and product publication versions in `PX-F/lib/catalog/types.ts`.
- Added frontend API client methods in `PX-F/lib/catalog/api.ts` for:
  - `GET /catalog/admin/products/{product_id}/snapshots`
  - `POST /catalog/admin/products/{product_id}/snapshots/{snapshot_id}/publish`
- Updated `PX-F/app/components/admin-product-edit-form.tsx` to load snapshots, track `published_version`, detect the latest unpublished generated snapshot, publish it, and refresh authoritative snapshot/media/snapshot state after publication.
- Updated `PX-F/app/components/product-editor/product-variants-section.tsx` with a localized publish-ready panel that appears only when `latestSnapshot.version > publishedVersion`.
- Added frontend tests for API routes, the publish-ready panel, and the integrated admin publish flow.
- No backend snapshot publication behavior, database schema, or backend API path was changed.
- Cleaned backend Ruff issues so the WSL backend lint gate passes.

## Current Architecture Decisions

- **Published snapshot owns storefront structure**: Once `product.published_version` is set, storefront structure reads must resolve from the matching immutable `CatalogProductStructureSnapshot`, not live draft options.
- **Published version owns storefront variants**: Storefront variant listing must remain scoped to variants generated for `product.published_version` and must not expose newer draft structures before publication.
- **Publication remains explicit**: Generation jobs create snapshots but do not publish them. Admins publish the chosen snapshot through the explicit publish endpoint.
- **Frontend publish visibility is version-driven**: The publish panel appears only when the latest generated snapshot version is newer than the product's published version.
- **Target-store policy wins**: Template application and structure copy must enforce the destination product/store tier, not the source/template origin.
- **Quota errors stay translatable at API boundaries**: Template quota failures must preserve the `AppException` envelope with `error.i18n_key` and interpolation `context`.
- **Rebuild media rebinding remains visible**: Full rebuilds that remove old variants must detach affected media and mark it for rebinding so admins can reconnect media to generated variants.
- **Block invalid structures early**: Public structure writes should refuse projected over-quota matrices before the user reaches rebuild/job execution.
- **Keep legacy safeguards**: Rebuild preview and job creation must still detect over-quota data that may come from migrations, imports, or older records.
- **Error envelopes stay i18n-compatible**: New user-facing policy failures should include `error.i18n_key` and `error.context`.

## Important Reasoning Behind Changes

Slice 16 focused on closing the product workflow gap between backend snapshot publication and admin usability. The backend already had immutable snapshots and publish endpoints, but the frontend could only view authoritative draft state. Wiring the existing endpoints into the variants tab makes the release path coherent without changing the underlying publication contract.

The implementation intentionally avoids auto-publish behavior. Storefront visibility must remain an explicit admin action so draft variant structure changes cannot leak into storefront responses.

The archive-vs-delete migration remains higher risk and was not changed in this slice.

## Pending Tasks

Still pending from the broader plan:
- Final archive-vs-delete migration strategy for variants and historical references.
- Broader browser-level E2E coverage for full admin setup -> generate -> publish -> storefront visibility.
- Investigate backend full-suite order sensitivity around `tests/test_store_domains.py::test_invalid_domain_returns_field_error`.
- Final dead-code cleanup after remaining design decisions settle.
- Optional deeper observability: alert policy surfaces, health views, reconnect metrics, and operational drilldowns.
- Keep planning docs aligned when future slices land.

## Known Issues or Blockers

- The backend venv in this workspace is Linux/WSL-style. From PowerShell, use `wsl bash -lc "cd /mnt/d/Github/muhsinmuhsy/PX/PX-B && .venv/bin/<command>"` for real backend validation. Direct PowerShell calls to `.venv\bin\python` can silently do nothing because the symlink/script layout is not native Windows.
- Backend full-suite validation is currently not boring: one run found an isolated-passing store-domain test failing only in the full suite, and a repeated full run timed out. Investigate this before treating backend full-suite health as clean.
- Some historical notes in `WHAT-DONE.md` describe earlier blocker work as it existed at that time. Treat those as history, not current instructions.
- Archive-vs-delete is still a real product/domain decision. Avoid hard-deleting historical variants further until that migration strategy is finalized.

## Important Files and Modules

Backend:
- `PX-B/app/modules/catalog/service.py`: Core variant business logic, snapshots, jobs, quota enforcement, selection resolution.
- `PX-B/app/modules/catalog/router.py`: Catalog admin/storefront API routes; Slice 16 only changed the metrics route query annotation for Ruff compliance.
- `PX-B/app/modules/catalog/variant_domain.py`: Shared domain constants and canonical identity helpers.
- `PX-B/app/modules/catalog/variant_events.py`: Durable SSE event bus.
- `PX-B/app/modules/catalog/worker.py`: Decoupled job worker and retry execution.
- `PX-B/app/exceptions/errors.py`: App exception classes and i18n error metadata.
- `PX-B/app/core/i18n_keys.py`: Backend i18n key registry.
- `PX-B/app/core/rate_limit/policies.py`: Rate-limit policy registry.

Frontend:
- `PX-F/lib/catalog/api.ts`: Catalog API client, now includes structure snapshot list/publish methods.
- `PX-F/lib/catalog/types.ts`: Frontend contract types, now includes snapshot/published version fields and `CatalogProductStructureSnapshotRead`.
- `PX-F/lib/catalog/admin-copy.ts`: Catalog admin UI copy, now includes snapshot publish keys.
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
- Frontend API client now calls existing backend endpoints:
  - `GET /catalog/admin/products/{product_id}/snapshots`
  - `POST /catalog/admin/products/{product_id}/snapshots/{snapshot_id}/publish`
- Frontend type contracts now expose `ProductRead.snapshot_version`, `ProductRead.published_version`, `CatalogVariantJobRead.snapshot_id`, and `CatalogProductStructureSnapshotRead`.
- No backend API path changes.
- No database migration.
- No backend schema changes.

Earlier completed work added job error tracking fields, durable variant job events, snapshots, metrics response types, and rate-limit policies. See `WHAT-DONE.md` for slice-by-slice details.

## Current Implementation Direction

The project should continue release hardening:
- Investigate and stabilize backend full-suite order sensitivity.
- Add browser-level E2E coverage for admin setup -> generate -> publish -> storefront visibility.
- Decide and implement archive-vs-delete migration.
- Clean stale/dead code after archive semantics settle.
- Keep quality gates explicit, and run backend gates through WSL in this workspace.

## Assumptions Made

- Unknown or missing store tier is treated as Basic.
- `pro` is the only higher tier currently recognized by `get_variant_limit_for_tier`.
- Public structure mutations should be prevented from creating over-quota matrices, even if generation is not immediately requested.
- Legacy/imported over-quota structures may still exist and must remain safely blocked at preview/job time.
- Snapshot publication remains explicit through `/catalog/admin/products/{product_id}/snapshots/{snapshot_id}/publish`; generation jobs do not auto-publish.
- The admin publish panel should be hidden unless a generated snapshot version is newer than the currently published storefront version.

## Recommended Next Steps

1. Investigate the backend full-suite `test_store_domains.py` order-sensitive failure and timeout.
2. Add browser-level E2E coverage for the complete admin generate/publish/storefront flow.
3. Plan the archive-vs-delete migration carefully before touching variant deletion semantics.
4. Keep plan/checklist docs current when Slices 16+ land.
5. Run full frontend gates and WSL backend gates after every implementation slice.

## Things Future Agents Should Avoid Breaking

- Do not bypass `assert_no_active_job` for structure mutations.
- Do not remove rebuild/job quota checks; they protect legacy/imported data.
- Do not hard-delete variants that may have historical references without the archive migration plan.
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
