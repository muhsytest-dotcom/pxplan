# Implementation Guide

Last updated: 2026-05-17

## Status

The original blocker implementation guide has been completed. The current work is release hardening after Slice 14.

Completed blocker areas:
- i18n framework and error envelopes.
- Worker error classification and retry.
- Job error tracking fields and migration.
- Backend and frontend tests for those behaviors.

Do not follow old instructions that say to install a separate frontend i18n framework, create blocker files from scratch, or start with Phase 1. Those are historical.

## Active Priority Matrix

| Priority | Work | Notes |
| --- | --- | --- |
| Critical | Archive-vs-delete migration strategy | Product/domain decision still required before changing deletion semantics. |
| High | Remaining E2E/API coverage | Published snapshot visibility is covered; template quota, media rebinding, and full browser flow remain. |
| Medium | Dead-code cleanup | Do after archive semantics settle. |
| Optional | Deeper observability | Alerting, health views, reconnect metrics, operational drilldowns. |
| Documentation | Keep guides aligned | Stale phase docs must not mislead future agents. |

## Implementation Standards

- Review live code before editing.
- Keep changes scoped to one slice.
- Use existing service/router/schema/test patterns.
- Maintain backward-compatible API shapes unless an explicit migration says otherwise.
- Preserve i18n metadata on user-facing errors.
- Add tests at the right level for every behavior change.
- Run backend and frontend gates before documenting completion.

## Backend Quality Gate

```powershell
cd PX-B
.\.venv\bin\python -m pytest tests -q
.\.venv\bin\ruff check .
.\.venv\bin\python -m mypy app
```

## Frontend Quality Gate

```powershell
cd PX-F
npm test
npm run lint
npm run typecheck
npm run i18n:check
npm run build
```

## Current Key Files

Backend:
- `PX-B/app/modules/catalog/service.py`
- `PX-B/app/modules/catalog/router.py`
- `PX-B/app/modules/catalog/models.py`
- `PX-B/app/modules/catalog/job_errors.py`
- `PX-B/app/modules/catalog/variant_events.py`
- `PX-B/app/modules/catalog/worker.py`
- `PX-B/app/core/i18n_keys.py`

Frontend:
- `PX-F/lib/catalog/api.ts`
- `PX-F/lib/catalog/types.ts`
- `PX-F/lib/catalog/i18n.ts`
- `PX-F/lib/catalog/variant-job-events.ts`
- `PX-F/app/components/admin-product-edit-form.tsx`
- `PX-F/app/components/product-editor/product-variants-section.tsx`

## Completion Documentation

After a slice is implemented and verified:
- Update `WHAT-DONE.md`.
- Update `AGENT_HANDOFF.md`.
- Include validation commands and results.
- Note any assumptions and things future agents must avoid breaking.
