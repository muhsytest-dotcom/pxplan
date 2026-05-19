# Current Work Notes

Last updated: 2026-05-19

## Current Direction

All core variants stabilization, asynchronous workers, snapshot publishing, and soft-delete archiving workflows are 100% complete through **Slice 25**. The workspace is extremely stable and ready.

Recommended next steps:
1. **Richer E2E/API Coverage**:
   - Unified admin-storefront browser-level integration flow: options setup -> generate -> publish -> storefront query.
   - Template apply quota failures and media rebinding end-to-end browser assertions.
2. **Advanced Observability**:
   - Add Prometheus alerting policies for SSE reconnection rates.
3. **Dead-Code Cleanup**:
   - Clean up any leftover helper files or legacy mock classes that became obsolete once the logical soft-delete archiving patterns finalized.

## Important Context for Future Slices

- Do not bypass `assert_no_active_job` on structure writes.
- Do not make storefront variant listing ignore `product.published_version`.
- Do not hard-delete variants that may have orders, snapshots, or media referencing them; direct variant removal must remain soft-deleted.
- Run both backend tests and frontend tests inside the Postgres-only environment after making modifications.

## Documentation Rule

After completing any new slice, always update `WHAT-DONE.md` and `AGENT_HANDOFF.md` before concluding the handoff.
