# Variants Stabilization Progress

Last updated: 2026-05-15

## Completed Slice 1: Core Domain Foundation

Implemented a shared variants domain foundation across backend and frontend.

Backend:
- Added `app/modules/catalog/variant_domain.py`.
- Added canonical variant identity helpers:
  - trim/case-normalize option value IDs
  - sort IDs deterministically
  - reject empty and duplicate value sets
  - generate deterministic SHA-256 identity hashes
- Added structure lifecycle values: `idle`, `dirty`, `synced`.
- Added variant lifecycle contract values: `active`, `archived`, `deleted`.
- Added centralized active/terminal job status guards.
- Added `timeout` to backend job status and phase enums.
- Wired variant generation to use the shared canonical key builder.
- Replaced duplicated active-job status checks with shared guard constants.

Frontend:
- Added `lib/catalog/variant-domain.ts`.
- Added matching canonical variant identity helpers.
- Added matching lifecycle/status types, including `timeout`.
- Updated `variant-matrix.ts` to use the shared canonical key logic.
- Updated variant action blocking to respect active queued/running jobs consistently.

Tests:
- Added backend domain tests in `tests/test_catalog_variant_domain.py`.
- Added frontend domain tests in `lib/catalog/__tests__/variant-domain.test.ts`.

## Completed Slice 2: SSE Job Event Infrastructure

Implemented the first production-safe SSE infrastructure layer while keeping the existing polling flow as fallback.

Backend:
- Added `app/modules/catalog/variant_events.py`.
- Added `VariantJobEventBus` with:
  - monotonic sequence numbers per job
  - replayable in-memory event history
  - typed event names: `state`, `completed`, `failed`, `cancelled`
  - SSE formatting with `id`, `event`, and JSON `data`
  - heartbeat support
  - idle timeout support
- Published job events from the existing job lifecycle update path.
- Added authenticated admin SSE endpoint:
  - `GET /catalog/admin/variant-jobs/{job_id}/events`
  - supports `last_event_id` replay cursor
  - validates store ownership before streaming
  - returns `text/event-stream`
  - sends `Cache-Control: no-cache` and `X-Accel-Buffering: no`

Frontend:
- Added `lib/catalog/variant-job-events.ts`.
- Added event stream URL builder.
- Added typed event payload parser.
- Added `EventSource` opener with credentials enabled.
- Added typed event payloads to `lib/catalog/types.ts`.

Tests:
- Added backend event bus tests in `tests/test_catalog_variant_events.py`.
- Added frontend event helper tests in `lib/catalog/__tests__/variant-job-events.test.ts`.

## Verification Completed

Backend:
- Full backend tests: `214 passed`
- Variant/snapshot regression suite: `43 passed`
- Backend domain/event focused tests: `9 passed`
- Full backend ruff: passed
- Full backend mypy: `83 source files`, passed

Frontend:
- Full frontend tests: `59 files / 291 tests passed`
- Frontend domain/event/matrix focused tests: `11 passed`
- Frontend lint: passed
- Frontend typecheck: passed
- Frontend production build: passed

## Still Not Complete

The full multi-phase plan is not finished yet. Remaining major areas:
- Dedicated worker process boundary and queue abstraction.
- Redis/pub-sub implementation for distributed SSE/event delivery.
- Durable job event storage or durable replay beyond in-memory history.
- Versioned structure snapshot tables and atomic storefront publication swap.
- Archive-vs-delete migration for existing variants and historical references.
- Full quota/tier/backpressure enforcement.
- Job cancellation API and timeout executor enforcement.
- Observability metrics, audit events, and recovery tooling.
- Full UI integration for live SSE progress with polling fallback.
- Broader E2E coverage for critical admin and storefront workflows.
- Broad dead-code cleanup across both apps after behavior stabilization.

## i18n Notes

No new user-facing copy was added in these slices. The frontend additions are infrastructure utilities only, so no translation keys were required.
