# Implementation Checklist

Last updated: 2026-05-17

## Current Checklist

Use this for new work after Slice 14. The old blocker checklist is complete.

### Before Coding

- [ ] Read `AGENT_HANDOFF.md`.
- [ ] Read `WHAT-DONE.md`.
- [ ] Confirm the requested work is not already complete.
- [ ] Review live code for existing patterns.
- [ ] Identify whether backend, frontend, database, API, or docs are affected.
- [ ] Decide the smallest production-ready slice.

### While Coding

- [ ] Preserve existing architecture.
- [ ] Avoid broad refactors unless necessary.
- [ ] Keep i18n-compatible errors.
- [ ] Preserve tenant/store ownership checks.
- [ ] Preserve active-job guards for structure mutations.
- [ ] Preserve published snapshot storefront boundaries.
- [ ] Add or update tests for changed behavior.
- [ ] Remove only truly unused code.

### Backend Validation

- [ ] `cd PX-B`
- [ ] `.\.venv\bin\python -m pytest tests -q`
- [ ] `.\.venv\bin\ruff check .`
- [ ] `.\.venv\bin\python -m mypy app`

### Frontend Validation

- [ ] `cd PX-F`
- [ ] `npm test`
- [ ] `npm run lint`
- [ ] `npm run typecheck`
- [ ] `npm run i18n:check`
- [ ] `npm run build`

### Documentation

- [ ] Update `WHAT-DONE.md` only after implementation and validation pass.
- [ ] Update `AGENT_HANDOFF.md` or a dedicated handoff file.
- [ ] Include latest status, completed work, reasoning, architecture decisions, pending tasks, known issues, changed files, API/database/schema changes, assumptions, next steps, and avoid-breaking notes.

## Completed Checklist Items

These are done and should not be reimplemented:
- [x] Backend i18n key registry.
- [x] AppException error envelope metadata.
- [x] Job error classification.
- [x] Worker retry/backoff.
- [x] Job error tracking database fields and migration.
- [x] Durable SSE events and replay.
- [x] Snapshot capture and explicit publish.
- [x] Metrics endpoint and dashboard.
- [x] SSE recovery fallback.
- [x] Batched variant selection resolution.
- [x] Target-store tier enforcement.
- [x] Published snapshot storefront regression coverage.

## Remaining Checklist Candidates

- [ ] Archive-vs-delete migration.
- [ ] Template apply/quota failure coverage.
- [ ] Media rebinding after rebuild coverage.
- [ ] Full browser-level admin/storefront E2E.
- [ ] Final dead-code cleanup.
- [ ] Optional observability drilldowns.
