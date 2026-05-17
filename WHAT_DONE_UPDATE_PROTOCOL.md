# WHAT-DONE.md Update Protocol

Last updated: 2026-05-17

## Rule

Update `WHAT-DONE.md` only after the implementation slice is complete and validated.

Do not update it:
- while planning;
- during partial implementation;
- before tests or checks run;
- when work is still uncertain;
- with aspirational or incomplete status.

Update it after:
- implementation is complete;
- relevant tests pass;
- lint and type checks pass;
- build/i18n checks pass when relevant;
- the work is ready for handoff.

## Current Slice Format

Use this shape:

```markdown
## Completed Slice N: Title

Status: Complete on YYYY-MM-DD

Summary:
- ...

Backend:
- ...

Frontend:
- ...

Tests:
- ...

Important reasoning:
- ...

API, database, and schema changes:
- ...

Verification completed:
- ...

Pending follow-up:
- ...
```

Include only sections that apply.

## Handoff Pairing

When `WHAT-DONE.md` is updated, also update `AGENT_HANDOFF.md` or another dedicated handoff file with:
- latest implementation status;
- summary of completed work;
- architecture decisions;
- reasoning;
- pending tasks;
- known issues or blockers;
- important files and modules;
- API/database/schema changes;
- assumptions;
- recommended next steps;
- things future agents must avoid breaking.

## Current Validation Commands

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
