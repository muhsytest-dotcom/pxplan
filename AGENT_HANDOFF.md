# Agent Handoff and Continuity

Last updated: 2026-05-17

Read this first, then `00_START_HERE.md`, `WHAT-DONE.md`, and the architecture docs. Some older guide files still describe the pre-implementation state, so treat this file plus `WHAT-DONE.md` and the live code as the current source of truth.

## Latest Implementation Status

The variants stabilization work is largely complete through Slice 13.

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

Slice 13 closed the tier-policy gap in structure writes:
- Added `ProductVariantTierQuotaExceededError` in `PX-B/app/exceptions/errors.py`.
- Added shared projected variant count and tier quota helpers in `PX-B/app/modules/catalog/service.py`.
- Enforced target-store Basic/Pro limits in:
  - `create_product_option_value_row`
  - `apply_variant_template_to_product`
  - `copy_product_structure`
- Preserved rebuild-impact and job-submission protection for legacy/imported over-quota structures.
- Added `PX-B/tests/test_catalog_tier_policy.py`.
- Updated `PX-B/tests/test_catalog_variant_ops.py` so the legacy over-quota regression seeds data directly instead of using now-protected public writes.

## Current Architecture Decisions

- **Target-store policy wins**: Template application and structure copy must enforce the destination product/store tier, not the source/template origin.
- **Block invalid structures early**: Public structure writes should refuse projected over-quota matrices before the user reaches rebuild/job execution.
- **Keep legacy safeguards**: Rebuild preview and job creation must still detect over-quota data that may come from migrations, imports, or older records.
- **Error envelopes stay i18n-compatible**: New user-facing policy failures should include `error.i18n_key` and `error.context`.
- **No broad refactors during policy work**: The latest slice intentionally stayed in existing service/router/test patterns.

## Important Reasoning Behind Changes

Before Slice 13, a Basic-tier store could build an oversized option matrix through direct option-value additions, template application, or structure copy. The rebuild preview and job endpoints would later block generation, but the structure itself could already be in a bad state. The new checks move policy enforcement to the mutation boundary while retaining downstream guards for old or imported data.

## Pending Tasks

Still pending from the broader plan:
- Final archive-vs-delete migration strategy for variants and historical references.
- Broader E2E coverage for critical admin and storefront flows.
- Final dead-code cleanup after remaining design decisions settle.
- Optional deeper observability: alert policy surfaces, health views, reconnect metrics, and operational drilldowns.
- Refresh stale planning docs. Several guide files still say error classification/retry/i18n are not implemented, but they are complete in the codebase and `WHAT-DONE.md`.

## Known Issues or Blockers

- Documentation drift exists in older plan files (`CODE_AGENT_COMPREHENSIVE_GUIDE.md`, `QUICK_REFERENCE.md`, `IMPLEMENTATION_CHECKLIST.md`, and related guide docs). Do not redo completed blocker work just because those files still show unchecked tasks.
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
- Error response semantics expanded for tier quota failures through existing `AppException` envelope:
  - `code`: `PRODUCT_VARIANT_TIER_QUOTA_EXCEEDED`
  - `error.i18n_key`: `catalog.variant.errors.tier_quota_exceeded`
  - `error.context`: `{ tier, limit, projected }`

Earlier completed work added job error tracking fields, durable variant job events, snapshots, metrics response types, and rate-limit policies. See `WHAT-DONE.md` for slice-by-slice details.

## Current Implementation Direction

The project should now move from stabilization into release hardening:
- Decide and implement archive-vs-delete migration.
- Add E2E tests around the most valuable flows.
- Clean stale docs and dead code.
- Keep quality gates full and boring: backend tests/lint/mypy; frontend tests/lint/typecheck/i18n/build.

## Assumptions Made

- Unknown or missing store tier is treated as Basic.
- `pro` is the only higher tier currently recognized by `get_variant_limit_for_tier`.
- Public structure mutations should be prevented from creating over-quota matrices, even if generation is not immediately requested.
- Legacy/imported over-quota structures may still exist and must remain safely blocked at preview/job time.

## Recommended Next Steps

1. Refresh stale plan/checklist docs so they reflect Slices 1-13 and stop misleading future agents.
2. Plan the archive-vs-delete migration carefully before touching variant deletion semantics.
3. Add E2E coverage for:
   - Variant setup and generate-missing.
   - Template apply and quota failure.
   - Media rebinding after rebuild.
   - Storefront published snapshot visibility.
4. Run the full quality gate after every implementation slice.

## Things Future Agents Should Avoid Breaking

- Do not bypass `assert_no_active_job` for structure mutations.
- Do not remove rebuild/job quota checks; they protect legacy/imported data.
- Do not hard-delete variants that may have historical references without the archive migration plan.
- Do not change error envelope shape; frontend error/i18n handling depends on it.
- Do not reintroduce per-variant selection N+1 loading in list endpoints.
- Do not make SSE execution perform work; SSE streams status only.
- Do not update `WHAT-DONE.md` until implementation and validation for a slice are complete.
