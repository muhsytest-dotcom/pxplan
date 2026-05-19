# Start Here

Last updated: 2026-05-19

## Purpose

This folder guides code agents working on the PX project. It is fully aligned with the live implementation through **Slice 25**.

## Current Status

The variants stabilization and logical soft-delete archiving work is complete through **Slice 25**. The entire backend and frontend test suites are 100% green under PostgreSQL with zero warnings.

### Key Stabilization Achievements:
- Core variant domain foundation and canonical identity.
- Durable SSE job events with automatic polling fallback.
- Variant explosion protection and Basic/Pro store tier limits.
- Decoupled worker process pool execution with cooperative cancellation.
- Immutable structure snapshots and explicit publication swaps.
- Custom auditing engine tracking logical archiving reasons and operators.
- Active-only variant uniqueness indexes allowing archived SKU restoration.
- PostgreSQL-only test harness and env isolation (Testcontainers + Alembic).

## Required Reading Order

1. `AGENT_HANDOFF.md` ← **Read this first!** It contains the core guidelines to prevent breaking features.
2. `WHAT-DONE.md` ← Detailed slice-by-slice changelog of what was implemented.
3. Relevant architecture specs:
   - `variant-domain-contract.md`
   - `technical-architecture-spec.md`
   - `ARCHIVE_DELETE_MIGRATION_STRATEGY.md`
4. `DO_NOT_FORGET.md` ← Read before finishing any new feature code.

## Source of Truth

Use this priority when documents disagree:
1. Live code.
2. `AGENT_HANDOFF.md` (Updated 2026-05-19).
3. `WHAT-DONE.md` (Updated with Slice 25).
4. Architecture contracts.

## Current Work / Pending Tasks

- **Richer E2E/API Coverage**:
  - Unified admin-storefront flow: options setup -> generate -> publish -> storefront query.
  - Template quota and media detaching end-to-end browser-level assertions.
- **Advanced Observability**:
  - setup Prometheus alerting policies for SSE reconnection rates.
- **Dead-Code Cleanup**:
  - Remove deprecated mock or legacy utility files once logical archiving is fully deployed to production.

## Validation Commands

All operations and tests run under PostgreSQL. SQLite is completely deprecated.

### Backend Validation:
```powershell
cd PX-B
python -m dotenv -f .env.local.dev run -- pytest -v
ruff check .
mypy app
```

### Frontend Validation:
```powershell
cd PX-F
npm test
npm run lint
npm run typecheck
npm run build
```
