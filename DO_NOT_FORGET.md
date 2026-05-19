# Do Not Forget

Last updated: 2026-05-19

## Current Reality

The core variant stabilization, snap publishing, and logical soft-deletions/archiving features are complete and verified through **Slice 25**. This checklist is to ensure any new feature work maintains these high architectural standards.

## Before Coding

- [ ] Read `AGENT_HANDOFF.md` completely.
- [ ] Read `WHAT-DONE.md`.
- [ ] Confirm the work is not already implemented.
- [ ] Review live code for active patterns.
- [ ] Identify tests that should change or be added.
- [ ] Decide whether database/API/schema behavior changes are required.

## Architecture Must-Preserve

- [ ] **Lock Out Structure Changes**: All option/value structure modifications must reject execution if an active variant job is found for that product.
- [ ] **Matrix Limits Enforced**: Prevent matrix explosion early on option writes, templates, and copy flows.
- [ ] **Published Snapshot Isolation**: Public storefront reads must always resolve against the immutable snapshot linked to `product.published_version`, protecting storefront displays from raw merchant drafts.
- [ ] **SSE Telemetry Only**: SSE handlers are strictly one-way progress monitors; they must never run business operations or database changes directly.
- [ ] **Error Envelope Integrity**: API validation checks must return standard Exception responses with `error.i18n_key` and `error.context`.
- [ ] **Soft-Delete by Default**: Variant removals must archive variants logically (`is_archived = true` and `archived_at` timestamped), never hard-delete them.
- [ ] **Active-Only uniqueness**: SKUs and option combinations are conditional on `is_archived = false`, allowing replacement variants to claim previously archived identifiers.
- [ ] **FK Clean Ordering**: When option value rows are removed, the variant values referencing them must be cleaned up before removing the option value row itself to stay PostgreSQL foreign-key compliant.

## Critical Archive-vs-Delete Rules

✅ **MUST DO**:
- Convert all hard deletes to soft-delete archive updates.
- Preserve variants with FK references (orders, snapshots, media) forever.
- Log all archive/restore events using the unified audit logging engine.
- Ensure storefront filters ignore archived variants while history remains queryable.

❌ **MUST NOT DO**:
- Hard-delete variants linked to storefront elements or snapshots.
- Hard-delete archived variants unless they are older than 90+ days and completely unreferenced.
- *Note for future Orders/Carts*: If orders and carts are added in the future, the cleanup policy **must immediately** check that the variant is not referenced by any order or cart.

## Validation

Everything must run under PostgreSQL. SQLite is deprecated.

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

## Documentation Finish

- [ ] Update `WHAT-DONE.md` only after all validation checks pass.
- [ ] Update `AGENT_HANDOFF.md` with latest details, changed files, decisions, changed schemas, and safety guidelines.
