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
- [ ] **Archive-by-default**: All variant deletion must use `is_archived = true`, not hard delete. See `ARCHIVE_DELETE_MIGRATION_STRATEGY.md` for complete specification. Never hard-delete variants linked to orders, snapshots, or media.

## Critical Archive-vs-Delete Rules

✅ **MUST DO**:
- Convert all hard deletes to archive updates
- Preserve variants with FK references forever
- Add `archived_at` timestamps
- Filter archived from storefront
- Log all archive/delete actions with audit trail
- Test that orders remain resolvable after archiving

❌ **MUST NOT DO**:
- Hard-delete variants referenced by orders
- Cascade-delete variants from snapshots
- Remove variants from published snapshots
- Skip archive logging
- Change deletion behavior without updating all FK surfaces

See `ARCHIVE_DELETE_MIGRATION_STRATEGY.md` before making any deletion changes.

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
