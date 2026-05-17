# Current Work Notes

Last updated: 2026-05-17

## Current Direction

PX variants are complete through Slice 14. Continue with release hardening.

Recommended next slices:
- Archive-vs-delete migration strategy and implementation.
- Remaining E2E/API coverage for template quota failures, media rebinding after rebuild, and full browser admin-to-storefront flow.
- Final dead-code cleanup after archive semantics settle.
- Optional deeper observability.

## Important Context

- Do not redo completed blocker work.
- Do not change variant hard-delete behavior without the archive migration plan.
- Storefront structure must stay pinned to published snapshots.
- Storefront variants must stay scoped to `product.published_version`.
- Keep backend and frontend i18n behavior aligned with the existing implementation.

## Documentation Rule

After completing a slice, update `WHAT-DONE.md` and `AGENT_HANDOFF.md`.
