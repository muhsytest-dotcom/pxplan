# Complete Guide System

Last updated: 2026-05-17

## Purpose

This folder is the continuity layer for agents working on PX. It must reflect the live system, not the original implementation backlog.

## Current Flow

```text
AGENT_HANDOFF.md
  -> WHAT-DONE.md
  -> 00_START_HERE.md
  -> CODE_AGENT_COMPREHENSIVE_GUIDE.md
  -> QUICK_REFERENCE.md
  -> DO_NOT_FORGET.md before finishing
```

## Current Status

Complete through Slice 14:
- Variant domain foundation.
- SSE durable events.
- Tier quotas.
- Worker classification/retry/cancellation/timeout.
- Snapshot capture and explicit publish.
- Backend/frontend i18n integration.
- Metrics and dashboard.
- SSE fallback.
- Batched selection resolution.
- Target-store tier enforcement.
- Published snapshot storefront regression coverage.

## Active Work Categories

- Archive-vs-delete migration.
- Remaining E2E/API coverage.
- Dead-code cleanup.
- Optional observability.
- Documentation alignment.

## Validation Gates

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

## Maintenance Rule

When implementation status changes, update:
- `WHAT-DONE.md`
- `AGENT_HANDOFF.md`
- Any guide that would otherwise send agents toward stale work.
