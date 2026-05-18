# Start Here

Last updated: 2026-05-17

## Purpose

This folder guides code agents working on the PX project. It is now aligned with the live implementation through Slice 14.

## Current Status

The variants stabilization work is complete through Slice 14. The old blocker roadmap is complete and should not be restarted.

Completed:
- Core variant domain foundation and canonical identity.
- Durable SSE job events with replay.
- Variant explosion protection and tier quotas.
- Decoupled worker execution with error classification, retry tracking, timeout, and cancellation.
- Immutable structure snapshots and explicit publication swaps.
- Backend i18n error envelopes and frontend catalog error-key translation.
- Variant job metrics and dashboard panel.
- Frontend SSE recovery fallback.
- Batched selection resolution for admin/storefront listings.
- Target-store tier enforcement for structure writes.
- Published snapshot storefront regression coverage.

## Required Reading Order

1. `AGENT_HANDOFF.md`
2. `WHAT-DONE.md`
3. `CODE_AGENT_QUICKSTART.md`
4. `CODE_AGENT_COMPREHENSIVE_GUIDE.md`
5. **`ARCHIVE_DELETE_MIGRATION_STRATEGY.md`** ← Read before any deletion changes
6. `QUICK_REFERENCE.md`
7. Relevant architecture references:
   - `variant-domain-contract.md`
   - `technical-architecture-spec.md`
   - `VARIANTS_FEATURE_STRUCTURE.md`
8. `DO_NOT_FORGET.md` before finishing

## Source Of Truth

Use this priority when documents disagree:

1. Live code.
2. `AGENT_HANDOFF.md`.
3. `WHAT-DONE.md`.
4. Architecture contracts.
5. Other guides.

## Current Work To Pick From

- **Archive-vs-delete migration strategy and implementation** ← Decision finalized, see `ARCHIVE_DELETE_MIGRATION_STRATEGY.md`
- Remaining E2E/API coverage:
  - template apply and quota failure;
  - media rebinding after rebuild;
  - full browser-level admin setup -> generate -> publish -> storefront visibility.
- Final dead-code cleanup after archive semantics settle.
- Optional deeper observability: alert policies, health views, reconnect metrics, and operational drilldowns.

## Do Not Pick

These were already implemented:
- error classification;
- worker retry;
- backend i18n key registry;
- frontend catalog error-key translation;
- job error tracking fields;
- durable SSE;
- snapshot capture;
- metrics dashboard;
- SSE fallback;
- batched variant selection resolution;
- target-store tier enforcement.

## Current Project Paths

- Backend: `PX-B`
- Frontend: `PX-F`
- Planning docs: `pxplan`

## Validation Commands

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

## Documentation Finish

After implementation and validation:
- Update `WHAT-DONE.md`.
- Update `AGENT_HANDOFF.md` or a dedicated handoff file.
- Include current status, completed work, decisions, reasoning, changed files, API/database/schema changes, assumptions, pending work, known issues, next steps, and avoid-breaking notes.
