# Guides Index

Last updated: 2026-05-17

## Source Of Truth

Use these first:
- `AGENT_HANDOFF.md`
- `WHAT-DONE.md`
- `00_START_HERE.md`
- `CODE_AGENT_COMPREHENSIVE_GUIDE.md`
- `QUICK_REFERENCE.md`
- `DO_NOT_FORGET.md`

## Architecture References

- `variant-domain-contract.md`
- `technical-architecture-spec.md`
- `VARIANTS_FEATURE_STRUCTURE.md`
- `plan.md`

## Historical Implementation References

These describe work that has already been implemented and are now reference-only:
- `ERROR_CLASSIFICATION.md`
- `I18N_IMPLEMENTATION.md`
- `IMPLEMENTATION_GUIDE.md`
- `IMPLEMENTATION_CHECKLIST.md`
- `CODE_AGENT_QUICKSTART.md`
- `GUIDES_CREATED.md`
- `_NEW_GUIDES_SUMMARY.md`
- `COMPLETE_GUIDE_SYSTEM.md`

The blocker phase content has been retired. Do not use historical docs to recreate completed systems.

## Current Status Summary

Completed through Slice 14:
- Domain foundation.
- Durable SSE.
- Tier quotas.
- Worker classification/retry/cancellation/timeout.
- Snapshot capture and explicit publication.
- i18n error envelopes and frontend catalog translation.
- Metrics and dashboard.
- SSE fallback.
- Batched selection resolution.
- Target-store tier enforcement.
- Published snapshot storefront regression coverage.

## Current Open Work

- Archive-vs-delete migration.
- Remaining E2E/API coverage.
- Dead-code cleanup.
- Optional deeper observability.

## Validation

Backend: `.\.venv\bin\python -m pytest tests -q`, `.\.venv\bin\ruff check .`, `.\.venv\bin\python -m mypy app`

Frontend: `npm test`, `npm run lint`, `npm run typecheck`, `npm run i18n:check`, `npm run build`
