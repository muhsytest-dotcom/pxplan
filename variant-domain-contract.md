# Variant Domain Contract

Status note, 2026-05-17:
This remains the business-rule contract for variants. For current implementation status and pending work, read `AGENT_HANDOFF.md` and `WHAT-DONE.md` first.

## Purpose
This document defines the authoritative business rules for variant lifecycle management in the ecommerce platform. It serves as the source of truth for behavior, superseding any implementation details.

## State Machines

### Structure Lifecycle
- **IDLE**: Clean, synced state with no pending changes
- **DIRTY**: Structure modified but not yet validated or generated
- **SYNCED**: Structure changes committed, variants generated and published to storefront

### Variant Job Lifecycle
- **QUEUED**: Job enqueued for execution
- **RUNNING**: Job actively processing
- **COMPLETED**: Job finished successfully
- **FAILED**: Job terminated with error
- **CANCELLED**: Job stopped by user or system
- **TIMEOUT**: Job exceeded execution time limit

## Variant Materialization Strategy
Deterministic rules for how structure changes affect variants:

- **Add Option Value**: Create new variants for all combinations including the new value
- **Remove Option Value**: Archive affected variants (never delete); archived variants remain resolvable for historical references (orders, carts, storefront links) but are excluded from active storefront discovery
- **Reorder Option**: No regeneration required (affects display only)
- **Rename Option/Label**: No structural change (affects presentation only)
- **Remove Option**: Invalidate and archive all affected variants
- **Change Option Type**: Full regeneration required

## Sync Semantics
Complete flow: Admin structure → validated → generated → published snapshot → storefront visible. Ensures atomic publication and prevents partial visibility.

## Publication Boundaries
- Storefront sees only fully published snapshots
- Publication is transactional (snapshot swap)
- If publication fails: previous published snapshot remains authoritative; failed snapshot never becomes visible
- Never expose half-generated or partially synced variants

## Lifecycle Guarantees
- Variants survive soft changes (labels, reordering)
- UUIDs remain stable across non-destructive changes
- Archive state preserves history for orders, analytics, refunds
- Hard changes trigger controlled regeneration

## Preservation Guarantees
Regeneration preserves:
- Inventory levels
- Media associations
- Pricing data
- SKU assignments
- External references (carts, analytics, SEO)

## Canonical Identity Rules
- Representation: Sorted optionValueIds string
- Ordering: Deterministic, locale-independent
- Normalization: Whitespace-trimmed, case-insensitive (display casing preserved separately)
- Hashing: SHA-256 for DB efficiency
- Uniqueness: Enforced at DB level

## Soft vs Hard Changes
- **Soft**: No regeneration (translations, labels, reordering)
- **Hard**: Regeneration required (add/remove options/values, type changes)

## Conflict Resolution
- Stale versions: Hard fail
- Active jobs: Reject mutations
- Structure changed during generation: Cancel job
- Duplicate requests: Idempotent handling

## Archive vs Delete Semantics
- **ACTIVE**: Live variants
- **ARCHIVED**: Preserved for history, not visible in storefront
- **DELETED**: Reserved for extreme cases only (admin override)

## Versioning
- structure_version: Tracks structural changes
- snapshot_version: Tracks generation snapshots
- published_version: Tracks storefront publication

## Snapshot Retention Policy
- Retain last 20 snapshots for rollback/debugging
- Retain published snapshots indefinitely
- Purge orphaned snapshots after 90 days

This contract must be strictly enforced in all implementations.
