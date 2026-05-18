# Archive-vs-Delete Migration Strategy (Critical)

**Status**: Product Decision Finalized & Documented
**Priority**: Critical — Agent Must Understand Before Any Deletion Changes

---

## Executive Summary

Modern production systems almost never truly delete important business entities anymore. This document establishes the PX variant retention strategy based on industry best practices proven at scale by companies like Shopify, Stripe, Notion, GitHub, and large ERP/e-commerce platforms.

**Decision**: Adopt **logical (soft) deletion** with archive states instead of physical deletion for all business-critical entities.

---

## Why Hard Delete Became Unpopular

### Distributed Inconsistency Risk

Modern systems are interconnected. A variant may appear in:
- Orders and invoices
- Analytics and exports
- Recommendation systems
- Caches and search indexes
- Event logs and audit trails
- Warehouse syncs and webhooks
- Customer support history
- Refund/chargeback processing

Hard deleting creates **distributed inconsistency** across all these surfaces.

### Storage Efficiency is No Longer a Driver

- Storage is cheap
- Broken history is expensive
- That changed industry behavior

10 years ago: Minimize storage.
**Now**: Preserve integrity.

---

## Modern Industry Pattern

### Why Soft Delete / Archive By Default

**Instead of:**
```sql
DELETE FROM variants WHERE id = ...
```

**Modern systems do:**
```sql
UPDATE variants
SET archived_at = now()
WHERE id = ...
```

Or:
```sql
UPDATE variants
SET is_deleted = true, deleted_at = now()
WHERE id = ...
```

### Industry Leaders' Approach

| Company | Model | Reason |
|---------|-------|--------|
| Shopify | Archive + Restore | Merchant recovery expectations |
| Stripe | Never delete payment records | Regulatory compliance |
| Notion | Trash bin + 30-day recovery | User experience |
| GitHub | Soft delete + archive | Historical issue tracking |
| Large ERPs | Audit-required archive | Financial audit trails |

---

## What Modern Systems Do: Active Preservation

### Typical State Machine

**Active State:**
- `is_archived = false`
- Visible everywhere
- Included in storefront
- Searchable
- Counted in limits

**Archived State:**
- `is_archived = true`
- Hidden from:
  - Storefront
  - Customer-facing search
  - Active discovery APIs
  - New order creation
- Still exists internally
- Historical references remain valid

**Why this works:**
- Orders still point to `variant_id = xxx` — **no broken FKs**
- No missing history
- Analytics remains intact
- Undo/restore support possible
- Event sourcing compatible

---

## What Modern Systems NEVER Hard-Delete

**Orders** — Never delete. Audit trail required.
**Payments** — Never delete. Regulatory compliance.
**Audit logs** — Never delete. Historical record.
**Published products/variants** — Archive instead. Restore expectation.

---

## What Modern Systems DO Hard-Delete (Carefully)

**Usually ONLY:**
- Temporary drafts
- Failed uploads
- Abandoned onboarding data
- Cache rows
- Expired sessions
- Orphan temporary entities (after TTL)

**NOT core business history.**

---

## Modern Retention Model: Proven Pattern

| Entity Type | Strategy | Reason |
|-------------|----------|--------|
| Orders | Never delete | Refunds, disputes, analytics |
| Payments | Never delete | Regulatory, audit, chargeback |
| Audit logs | Never delete | Legal, compliance |
| Published variants | Archive only | Restore, historical SKU refs |
| Draft variants | Cleanup later (90d+) | Temporary working state |
| Sessions | TTL delete | Security, not business-critical |
| Cache | Immediate delete | Rebuilds automatically |
| Media | Archive with variant | Photo history, rebinding |

---

## The Emerging Trend: Append-Only Systems

The industry trend is moving even further toward:
- **Event sourcing**: Immutable logs of all changes
- **Temporal tables**: Historical snapshots preserved at DB level
- **Audit-first systems**: Delete never, archive always

**Meaning:**
Instead of modifying/deleting history,
systems preserve historical truth permanently.

---

## Recommended Model for PX Variants

### Keep Forever
- Published variants
- Referenced variants (in orders, carts, exports)
- Snapshot-linked variants
- Audit-linked variants

### Allow Cleanup Only For
- Unused drafts (older than 90+ days)
- Orphan temporary entities
- Failed imports (never materialized)
- Incomplete rebuilds

### Never Hard-Delete
- Any variant linked to an order
- Any variant in published snapshots
- Any variant with price/inventory history
- Any variant with media
- Any archived variant with FK references

---

## PX Variant Lifecycle: Final Design

### States
```
ACTIVE  → is_archived = false
          Visible in storefront
          Active in new orders
          Discoverable

ARCHIVED → is_archived = true
          Hidden from storefront
          Not available for new orders
          Still queryable for history
          Reference-safe

DELETED → Exceptional override only
          Admin action with audit logging
          Only after manual verification
          Never automatic
```

### Transition Rules
```
ACTIVE  → ARCHIVED  (automatic on option/value removal)
ACTIVE  → ARCHIVED  (automatic on rebuild invalidation)
ARCHIVED → ACTIVE   (admin restore action)
ARCHIVED → DELETED  (extreme cases only, audit logged)
DELETED → (never restored, but record preserved for audit)
```

### Preservation Guarantees
When archiving, **preserve forever**:
- Inventory history
- Price history
- Media associations
- SKU assignments
- Order references
- Cart history
- External analytics links
- Customer view history

---

## Implementation Approach for PX

### Phase 1: Safety Baseline (Immediate)
1. Audit all hard-delete paths in `PX-B/app/modules/catalog/service.py`
2. Identify variants with FK references in:
   - Orders
   - Snapshots
   - Media
   - Tier policies
   - External syncs
3. Document current deletion triggers

### Phase 2: Archive State Implementation
1. Ensure `is_archived` or `archived_at` field exists on variants table
2. Update storefront queries to filter `WHERE is_archived = false`
3. Update active-variant counts to exclude archived
4. Add admin restore endpoint `/catalog/admin/variants/{id}/restore`

### Phase 3: Migration Path
1. Create migration: Convert hard-delete triggers to archive updates
2. Preserve all variant records that ever had FK references
3. Update indexes to include archive state in storefront queries
4. Add admin audit view for archived/deleted variants

### Phase 4: Cleanup Policy
```sql
-- Only safe to delete after these conditions:
-- 1. is_archived = true
-- 2. archived_at > 90 days ago
-- 3. No FK references in orders, carts, snapshots
-- 4. No media attachments
-- 5. No price/inventory history
-- 6. Not in any published snapshot

DELETE FROM variants 
WHERE is_archived = true 
  AND archived_at < NOW() - INTERVAL '90 days'
  AND NOT EXISTS (SELECT 1 FROM orders WHERE variant_id = variants.id)
  AND NOT EXISTS (SELECT 1 FROM carts WHERE variant_id = variants.id)
  -- ... etc for all FK surfaces
```

---

## What Agents Should NOT Do (Critical)

❌ **Do NOT** hard-delete variants without this migration strategy
❌ **Do NOT** cascade-delete variants from orders or published snapshots
❌ **Do NOT** remove variants from snapshots—recreate snapshot instead
❌ **Do NOT** restore hard-deleted variants—only archive can restore
❌ **Do NOT** skip audit logging when deleting/archiving
❌ **Do NOT** change deletion behavior without updating all FK surfaces first

---

## What Agents SHOULD Do (Required)

✅ **DO** convert all hard deletes to archive updates
✅ **DO** preserve variant records with FK references forever
✅ **DO** add `archived_at` timestamps to deleted records
✅ **DO** filter archived variants from storefront queries
✅ **DO** include archive state in admin audit logs
✅ **DO** test that orders remain resolvable after variant archiving
✅ **DO** verify published snapshots still resolve archived variants
✅ **DO** implement admin restore endpoint for archived variants
✅ **DO** document cleanup policy before any bulk operations
✅ **DO** treat archived SKU as permanently reserved (never reuse)
✅ **DO** preserve snapshot history—never mutate published snapshots

---

## Alignment with PX Architecture

Your snapshot architecture already aligns perfectly with this pattern:

**Snapshots are immutable.**
- They capture historical truth
- Published snapshots live forever
- Variants in published snapshots are preserved forever

**Draft variants can change.**
- But changes are tracked per snapshot_version
- Historical versions are preserved

**Archive states fit naturally.**
- Archived variants are excluded from active storefront (using `published_version`)
- But remain in historical snapshots
- Orders pointing to archived variants still resolve correctly

---

## Testing & Verification

Before and after implementing archive semantics:

```python
# Test 1: Archived variants are not visible in storefront
def test_archived_variants_excluded_from_storefront():
    variant = create_variant(is_archived=False)
    assert variant in storefront_variants()
    
    archive_variant(variant)
    assert variant not in storefront_variants()

# Test 2: Orders remain resolvable after archiving
def test_order_references_survive_variant_archiving():
    order = create_order(variant_id=variant.id)
    archive_variant(variant)
    
    assert order.variant_id == variant.id
    assert resolve_variant(variant.id) is not None

# Test 3: Published snapshots preserve archived variants
def test_published_snapshot_includes_archived_variants():
    snapshot = publish_snapshot(...)
    variant = variant_in_snapshot(snapshot)
    
    archive_variant(variant)
    assert variant_in_snapshot(snapshot) is not None

# Test 4: Media is rebound on rebuild, not deleted
def test_media_rebinding_after_rebuild():
    media = attach_media_to_variant(variant)
    rebuild_structure()
    assert media.status == 'NEEDS_REBINDING'
```

---

## Architectural Invariants (Critical for Agents)

These invariants must hold true at all times. If future code violates them, it creates data corruption.

**1. Published snapshots are immutable forever.**
- No hard delete of variants referenced by published snapshots.
- Snapshot references are immutable audit records.
- Violation: customer orders with variant references become dangling.

**2. Orders must always resolve archived variants.**
- Archive state change must never break FK links.
- Archived variants remain queryable for order fulfillment, refunds, and returns.
- Violation: impossible to process refunds or answer "what was in this order?".

**3. Archiving must never mutate historical snapshots.**
- Publishing a snapshot captures immutable state at that moment.
- Later archiving does not retroactively change published snapshots.
- Violation: audit trail becomes unreliable.

**4. Restore must preserve original variant identity.**
- Restoring a variant must reactivate the original record (preserving its UUID, SKU, and history) rather than cloning or creating a new record.
- Violation: database integrity breaks, orphan records accumulate, and audit trails lose track of the entity's history.

**5. Cleanup must never delete referenced entities.**
- The periodic cleanup task must never delete any variant that has active references to media, attributes, snapshots, or external domains.
- Violation: breaks referential integrity and deletes valuable business history.

**6. SKU is a write-once identity.**
- Never reuse archived SKU for a new active variant.
- Reused SKU causes: analytics confusion, warehouse sync corruption, refund misrouting.
- Violation: "sku-123 refund" becomes ambiguous (old or new product?).

**7. Archive state changes must be auditable.**
- Every archive/restore action logged with admin, timestamp, reason.
- Reason must be queryable for analytics/debugging (e.g., "OPTION_REMOVED", "MANUAL_ADMIN").
- Violation: impossible to explain why a variant disappeared.

**8. Media detachment must precede variant archiving.**
- When a variant is archived, its media must be marked for rebinding.
- Don't leave orphan media expecting archived variants.
- Violation: media stays invisible until orphan cleanup runs.

---

## Future Extensions (Not Required Now, High Value Later)

### 1. Legal Hold (Enterprise Compliance)

For industries with legal discovery/regulatory holds:

```python
class ProductVariant(Base):
    is_legal_hold: bool = False
    legal_hold_reason: str | None = None
    legal_hold_at: datetime | None = None
```

**Semantics**:
- Cannot archive or delete while `is_legal_hold = true`
- Blocks cleanup policy entirely
- Used for compliance, litigation holds, investigations

**When to add**: When first enterprise customer with legal discovery requirements signs.

### 2. Archive Reason Enums (Analytics & Debugging)

Instead of free-text only, capture structured reasons:

```python
class ArchiveReason(Enum):
    OPTION_REMOVED = "option_removed"           # Option/value was deleted
    REBUILD_INVALIDATED = "rebuild_invalidated" # Variant invalidated by rebuild
    MANUAL_ADMIN = "manual_admin"               # Admin explicitly archived
    DUPLICATE = "duplicate"                      # Merge/consolidation
    MERGED = "merged"                           # Variant merged into another
    IMPORT_FAILURE = "import_failure"           # Failed bulk import
    SKU_CONFLICT = "sku_conflict"              # SKU collision resolved via archive

class ProductVariant(Base):
    archive_reason: ArchiveReason | None = None
    archive_context: dict[str, Any] | None = None  # e.g., {"merge_target_id": "..."}
```

**Value**:
- Analytics: "80% of archives are rebuild invalidations—option hierarchy is confusing"
- Support tooling: "Show me all duplicates that were merged"
- Rollback analysis: "Restore all variants archived from failed import X"
- Debugging: "Why did this SKU become unavailable?"

**When to add**: After first few hundred archive events in production (enough data to justify).

### 3. Immutable Variant Audit Events Table

Instead of only mutating variant rows, append to an audit log:

```python
class VariantAuditEvent(Base):
    id: UUID
    product_id: UUID
    variant_id: UUID
    event_type: str  # ARCHIVED, RESTORED, PUBLISHED, INVALIDATED
    old_state: dict  # Previous values
    new_state: dict  # New values
    admin_id: UUID
    reason: str
    created_at: datetime
    snapshot_id: UUID | None  # For rebuild/publish events
```

**Value**:
- Audit trail for compliance/investigations
- Debugging: "Show me every change to variant 123 and why"
- Rollback analysis: "Restore to state X and reapply state Y differently"
- Support: "What changed between when order was placed and now?"

**When to add**: When audit compliance becomes mandatory (SOC 2, HIPAA, etc.)

### 4. SKU Reservation After Archive

Ensure archived SKU can never be reused:

```python
class ReservedSKU(Base):
    sku: str
    product_id: UUID
    archived_at: datetime
    reason: str  # e.g., "variant_archived"
    reserved_until: datetime | None  # Forever if None
```

**Semantics**:
- When variant with SKU is archived, add row to `ReservedSKU`
- On new variant creation, check if SKU in `ReservedSKU` for same product
- Reject: "SKU 'BLUE-XL' was archived on 2025-03-15 for this product. Pick a new SKU or contact support to unreserve."

**When to add**: After first SKU-reuse confusion bug in production.

---

## References

- Shopify's approach to product soft-deletion
- Stripe's immutable payment record design
- Enterprise GDPR compliance for data deletion
- Event sourcing best practices (Event Store)
- Temporal databases and historical tracking
- ecommerce industry standards for variant lifecycle

---

## Decision Ownership

**Decided**: Archive-by-default pattern for all business-critical entities
**Applies to**: Variants, options, values, media, pricing, inventory history
**Exception**: Temporary drafts older than 90 days (cleanup policy)
**Audit**: All archive/delete actions must be logged with admin, timestamp, reason

**Agents implementing this**: Follow the phases above. Update WHAT-DONE.md and AGENT_HANDOFF.md when complete.

---

## Next Steps for Agents

1. **Before touching deletion logic**: Read this document entirely
2. **Audit phase**: Identify all hard-delete paths
3. **Design phase**: Plan FK reference preservation
4. **Implementation phase**: Convert deletes to archives
5. **Testing phase**: Verify orders/snapshots still resolve
6. **Documentation**: Update variant-domain-contract.md with final states
7. **Handoff**: Update AGENT_HANDOFF.md with completion status

---

**Last Updated**: 2026-05-18
**Status**: Finalized — Agents must follow this strategy
