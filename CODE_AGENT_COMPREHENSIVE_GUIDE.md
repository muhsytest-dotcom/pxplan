# Code Agent Comprehensive Guide

Last updated: 2026-05-17

## Purpose

This is the current implementation guide for code agents working on PX variants after Slice 14. The old five-phase blocker roadmap has been completed and should not be re-executed.

## Current System State

Production stabilization is complete through Slice 14:
- Domain foundation and canonical identity.
- Durable SSE event stream with replay.
- Tier-aware explosion protection.
- Decoupled worker execution, retry, classification, cancellation, and timeout handling.
- Snapshot capture and explicit storefront publication.
- Backend i18n error envelopes and frontend catalog translation.
- Metrics, dashboard panel, and write rate limiting.
- SSE recovery fallback in frontend.
- Batched selection resolution to avoid listing N+1 behavior.
- Target-store tier enforcement for structure writes.
- Regression coverage for published snapshot storefront visibility.

## Source Of Truth Order

1. Live code.
2. `AGENT_HANDOFF.md`.
3. `WHAT-DONE.md`.
4. `variant-domain-contract.md`.
5. `technical-architecture-spec.md`.
6. This guide and `QUICK_REFERENCE.md`.

If this or any other document conflicts with live code plus the handoff, update the docs before coding.

## Current Release-Hardening Roadmap

### Slice A: Archive-vs-Delete Migration

**Status**: Decision finalized — See `ARCHIVE_DELETE_MIGRATION_STRATEGY.md` for complete specification.

Goal: implement the variant retention strategy following modern industry patterns.

**Decision basis** (proven at Shopify, Stripe, Notion, GitHub):
- Historical orders, carts, analytics, media, attributes, and external references must remain resolvable.
- Use logical (soft) deletion: `is_archived = true` instead of hard delete.
- `ARCHIVED` variants excluded from storefront but remain queryable for history and FK safety.
- `DELETED` reserved for exceptional admin override cases only (rare).
- Storage is cheap; broken history is expensive.

**Required work**:
1. Audit current hard-delete paths in `PX-B/app/modules/catalog/service.py` and repository helpers.
2. Identify all FK surfaces (orders, carts, snapshots, media, tier policies, external syncs).
3. Implement archive state in database migrations (add `is_archived`, `archived_at` fields).
4. Update storefront queries to filter `WHERE is_archived = false`.
5. Implement admin restore endpoint `/catalog/admin/variants/{id}/restore`.
6. Add focused backend tests for:
   - Option/value removal → archive affected variants
   - Rebuild orphan handling → archive, not delete
   - Direct variant removal → archive with audit log
   - Media rebinding → preservation after archiving
   - Storefront filtering → archived variants hidden but queryable
   - Order resolution → still resolve archived variants
7. Update published snapshot logic to preserve archived variants in snapshots.
8. Update admin audit views to show archive/delete history.

**Critical**: Read `ARCHIVE_DELETE_MIGRATION_STRATEGY.md` before making any deletion changes.

### Slice B: E2E/API Coverage

Goal: protect the critical admin-to-storefront workflows.

Already covered:
- Published snapshot visibility remains pinned while draft structure changes continue.

Still valuable:
- Template apply and quota failure.
- Media rebinding after rebuild.
- Full admin setup -> generate -> publish -> storefront visibility flow.
- Storefront published structure and variant localization.

### Slice C: Dead-Code Cleanup

Goal: remove genuinely unused code after domain decisions settle.

Rules:
- Do not delete compatibility paths that tests or clients still rely on.
- Do not remove error envelope metadata.
- Do not remove old variant guards that protect imported or legacy data.
- Keep cleanup isolated and fully validated.

### Slice D: Deeper Observability

Optional:
- Alert policy surfaces.
- Health views.
- SSE reconnect metrics.
- Operational drilldowns.

## Architecture Rules

- Storefront reads published snapshots when `product.published_version > 0`.
- Storefront variant listings must remain scoped to `product.published_version`.
- Generation jobs do not auto-publish; publication is explicit.
- SSE streams status only; execution belongs to workers.
- Structure mutations must reject active jobs.
- Public structure writes must enforce target-store Basic/Pro quotas.
- Rebuild preview and job creation must still protect legacy/imported over-quota structures.
- Backend user-facing errors must keep `error.i18n_key` and `error.context`.
- Frontend i18n must follow the existing locale/path copy system and catalog error-key helper.

## Validation Gates

Run the full relevant gate for every completed slice.

Backend:

```powershell
cd PX-B
.\.venv\bin\python -m pytest tests -q
.\.venv\bin\ruff check .
.\.venv\bin\python -m mypy app
```

Frontend:

```powershell
cd PX-F
npm test
npm run lint
npm run typecheck
npm run i18n:check
npm run build
```

## Documentation Rules

After implementation and validation:
- Update `WHAT-DONE.md`.
- Update `AGENT_HANDOFF.md` or another dedicated handoff file.
- Include completed work, reasoning, architecture decisions, changed files, validation, pending tasks, assumptions, and things future agents must not break.

Do not update `WHAT-DONE.md` before the implementation slice is complete.
