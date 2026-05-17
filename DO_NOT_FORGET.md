# Do Not Forget

Last updated: 2026-05-17

## Current Reality

The old production blockers are complete through Slice 14. This checklist is for new release-hardening work.

## Before Coding

- [ ] Read `AGENT_HANDOFF.md`.
- [ ] Read `WHAT-DONE.md`.
- [ ] Confirm the work is not already implemented.
- [ ] Review live code for patterns.
- [ ] Identify tests that should change or be added.
- [ ] Decide whether database/API/schema behavior changes are required.

## Architecture Must-Preserve

- [ ] Structure mutations reject active jobs.
- [ ] Rebuild preview and job creation still protect legacy/imported over-quota structures.
- [ ] Target-store tier policy remains enforced.
- [ ] Storefront structure uses the published snapshot when `published_version > 0`.
- [ ] Storefront variants are scoped to `published_version`.
- [ ] Generation jobs do not auto-publish.
- [ ] SSE streams status only.
- [ ] Error envelopes keep `error.i18n_key` and `error.context`.
- [ ] Tenant/store ownership checks remain in all admin paths.
- [ ] Batched selection resolution is not replaced with N+1 lookups.

## Current High-Risk Open Decision

Archive-vs-delete is unresolved. Do not expand hard-delete behavior for variants that may have historical references until the migration strategy is designed and documented.

## Validation

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

- [ ] Update `WHAT-DONE.md` only after validation passes.
- [ ] Update `AGENT_HANDOFF.md` or a dedicated handoff file.
- [ ] Document completed work, reasoning, architecture decisions, pending tasks, known issues, changed files, API/database/schema changes, assumptions, next steps, and avoid-breaking notes.
