# Quick Reference

Last updated: 2026-05-17

## Current Position

PX variants are complete through Slice 14. The old Phase 1-5 blocker checklist is complete and no longer the active roadmap.

Use this card while coding.

## Read Order

1. `AGENT_HANDOFF.md`
2. `WHAT-DONE.md`
3. `00_START_HERE.md`
4. `CODE_AGENT_COMPREHENSIVE_GUIDE.md`
5. This file
6. `DO_NOT_FORGET.md` before finishing

## Do Next

Pick one scoped release-hardening slice:
- Archive-vs-delete migration plan and implementation.
- Template apply/quota failure coverage.
- Media rebinding after rebuild coverage.
- Full browser E2E for admin setup -> generate -> publish -> storefront.
- Dead-code cleanup after archive semantics are decided.
- Optional observability drilldowns.

## Do Not Redo

Already done:
- `app/modules/catalog/job_errors.py`
- `app/core/i18n_keys.py`
- job error DB fields and migration
- worker retry and classification
- backend i18n error envelopes
- frontend catalog error-key translation
- durable SSE replay
- snapshots and publish swaps
- metrics and dashboard panel
- SSE fallback polling
- batched selection resolution
- target-store tier enforcement
- published snapshot storefront regression coverage

## Commands

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

## Must Preserve

- `assert_no_active_job` on structure mutations.
- Rebuild/job quota checks for legacy/imported data.
- `error.i18n_key` and `error.context` envelopes.
- `product.published_version` as storefront variant boundary.
- Published snapshot structure reads when a snapshot exists.
- Batched selection loading in list endpoints.
- SSE status-only behavior.
- Explicit publish; generation jobs must not silently publish.

## Documentation Finish

Only after validation:
- Update `WHAT-DONE.md`.
- Update `AGENT_HANDOFF.md` or the current dedicated handoff.
- Document assumptions, decisions, files changed, validation, pending work, and avoid-breaking notes.
