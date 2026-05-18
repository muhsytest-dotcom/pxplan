# Code Agent Quickstart

Last updated: 2026-05-17

This guide reflects the live PX variants implementation through Slice 14. Older phase language in historical docs has been retired: error classification, worker retry, backend/frontend i18n, durable SSE, snapshots, observability, batched selection resolution, tier enforcement, and published-snapshot storefront coverage are already implemented.

## Current Status

The variants system is in release-hardening, not initial blocker implementation.

Completed:
- Core variant domain foundation and canonical identity.
- Durable SSE job events with replay.
- Variant explosion protection and Basic/Pro quotas.
- Decoupled worker boundary, error classification, retry tracking, timeout, and cancellation.
- Immutable product structure snapshots and explicit publication swaps.
- Backend i18n error envelopes and frontend catalog error-key translation.
- Variant job metrics, instrumentation, and dashboard panel.
- SSE recovery fallback with live/polling state in the admin UI.
- Batched selection resolution for admin and storefront listings.
- Target-store tier enforcement for option-value growth, template application, and structure copy.
- API-level regression coverage for published snapshot storefront visibility while draft structure changes continue.

## Read First

1. `AGENT_HANDOFF.md`
2. `WHAT-DONE.md`
3. `00_START_HERE.md`
4. `CODE_AGENT_COMPREHENSIVE_GUIDE.md`
5. **`ARCHIVE_DELETE_MIGRATION_STRATEGY.md`** ← Required before any deletion changes
6. `QUICK_REFERENCE.md`
7. `DO_NOT_FORGET.md` before finishing

Treat `AGENT_HANDOFF.md`, `WHAT-DONE.md`, and the live code as the source of truth when any older plan text conflicts.

## Recommended Next Work

Pick one release-hardening slice:
- **Implement archive-vs-delete migration** ← See `ARCHIVE_DELETE_MIGRATION_STRATEGY.md` (decision finalized)
- Add remaining E2E/API coverage for template quota failures, media rebinding after rebuild, and full browser admin-to-storefront flows.
- Clean stale/dead code after the archive strategy is decided.
- Add optional deeper observability such as alert policies, health views, reconnect metrics, and drilldowns.

## Current Commands

Backend, from `PX-B`:

```powershell
.\.venv\bin\python -m pytest tests -q
.\.venv\bin\ruff check .
.\.venv\bin\python -m mypy app
```

Frontend, from `PX-F`:

```powershell
npm test
npm run lint
npm run typecheck
npm run i18n:check
npm run build
```

## Important Rules

- Do not redo completed blocker work just because an archived guide mentions it.
- Do not introduce a second frontend i18n framework; the frontend currently uses the established locale/path copy system plus catalog error-key translation.
- Do not hard-delete variants with possible historical references until the archive-vs-delete strategy is finalized.
- Do not bypass `assert_no_active_job` for structure mutations.
- Do not make storefront structure read live draft options when a published snapshot exists.
- Do not make storefront variants ignore `product.published_version`.
- Keep all user-facing backend errors i18n-compatible through `error.i18n_key` and `error.context`.
- Update `WHAT-DONE.md` only after implementation and validation are complete.
