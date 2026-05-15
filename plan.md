# Variants Feature Architectural Stabilization Plan

## Overview
This enterprise-grade plan establishes a robust commerce domain architecture with deterministic behavior, operational resilience, async separation, concurrency modeling, scalability boundaries, and maintainability strategies for a serious FastAPI ecommerce platform.

## Core Architectural Principles
- **Domain-First**: Stabilize variant lifecycle before UI/UX improvements
- **Canonical Identity**: Single authoritative variant representation
- **Snapshot Safety**: Jobs execute against immutable structure snapshots
- **SSE Streaming**: Async job progress via Server-Sent Events
- **Eventually Consistent Client State**: Frontend reconciles with REST, accepts temporary lag
- **Eventually Consistent Storefront**: Admin saves do not immediately update storefront visibility
- **Async Worker Boundary**: SSE streams status, does not execute jobs
- **At-Least-Once Delivery**: SSE events may duplicate; frontend must tolerate this
- **Authoritative Sources of Truth**:
  - Structure: DB
  - Job Progress: Worker memory/pubsub
  - Storefront Variants: Synced published snapshot
  - UI Draft State: Frontend form state
  - Generation Status: Job lifecycle store

## Job Lifecycle State Machines
- **Structure Lifecycle**: IDLE, DIRTY, SYNCED
- **Job Lifecycle**: QUEUED, RUNNING, COMPLETED, FAILED, CANCELLED, TIMEOUT

## Sync Semantics
Admin structure → validated → generated → published snapshot → storefront visible (prevents partial visibility and inventory mismatches).

## Variant Generation Strategy & Materialization Rules
- **Incremental Diff-Aware**: Snapshot + diff-based generation (preserves existing variants).
- **Variant Materialization Strategy**:
  - Add value: Create new variants
  - Remove value: Archive affected variants
  - Reorder option: No regeneration
  - Rename label: No structural change
  - Remove option: Invalidate affected variants
- **Archive vs Delete Semantics**: Use ACTIVE, ARCHIVED, DELETED states (never hard-delete for order history, analytics, refunds).
- **Partial Failure Semantics**: Rollback all or partial commit with retry for failed subset.
- **Inventory & Media Preservation**: Regeneration preserves inventory, images, pricing, SKUs, references, analytics.
- **Stable Variant UUID**: UUID survives soft changes, label updates, reorderings.

## Storefront Publication Transaction Boundary
- **Atomic Publication**: Snapshot swap ensures storefront never sees half-published variants.
- **Versioned Structure Snapshots**: structure_version, snapshot_version, published_version for rollback, debugging, auditability.

## Operational Limits by Tier
- **Basic Plan**: 5K max variants
- **Pro Plan**: 50K max variants

## Security Boundaries
- Tenant isolation, store ownership validation, quota enforcement security, SSE auth expiration, signed event access for multi-tenant ecommerce.

## Migration Strategy
- Structure migration, UUID migration, canonical key migration, rollback plans for existing production data.

## Operational Modes
- **Normal**: Full generation
- **Degraded**: Reduced concurrency
- **Maintenance**: Generation disabled

## Implementation Phases (Architectural Order)

### Phase 1 — Stabilize Core Domain Model (MOST IMPORTANT)
**Goal**: Establish authoritative variant lifecycle rules.

**Actions**:
- **Formalize State Machines**: Separate structure and job lifecycle states.
- **Create Central Variant Capability Guards**: Implement functions.
- **Canonical Variant Identity Strategy**: Single canonical representation.
- **Soft vs Hard Structural Changes**: Hard changes require regeneration.
- **Document Variant Domain Contract**: Include rules and guarantees.

**Validation Gate**: Domain contract approved.

**Files**: models.py, schemas.py, use-product-variant-actions.ts

### Phase 2 — Concurrency & Transaction Safety
**Goal**: Deterministic conflict handling.

**Actions**:
- **Explicit Conflict Strategy**: Behaviors for versions, jobs, changes.
- **Snapshot-Based Job Execution**: Execute against immutable snapshots.
- **API & Worker Idempotency**: Keys for endpoints; guarantees for workers.
- **Transaction Retry Logic**: Retry mechanisms.
- **Retry Classification Rules**: SYSTEM_FAILED=yes, CONFLICT_FAILED=no, etc.

**Validation Gate**: Snapshot semantics finalized.

**Files**: router.py, service.py, schemas.py

### Phase 3 — Variant Explosion Protection
**Goal**: Prevent overload.

**Actions**:
- **3-Layer Protection**: Caps, detection, quotas.
- **Input Validation**: Enforce limits.
- **Resource Cost Awareness**: Estimates.
- **Backpressure Strategy**: Limits, prioritization, throttling.

**Validation Gate**: Protection tested.

**Files**: service.py, schemas.py, models.py

### Phase 4 — Backend Performance Architecture
**Goal**: Optimize large-scale operations.

**Actions**:
- **Split Preview vs Full Matrix APIs**: Lightweight vs paginated.
- **Database Optimizations**: Indexes, batching.
- **Connection Pool Config**: Settings.
- **Job Timeout Handling**: Cleanup.
- **Long-Running Job Isolation**: Separate worker pools.

**Validation Gate**: Benchmarks met.

**Files**: models.py, variant-matrix.ts, service.py

### Phase 5 — Frontend State Architecture
**Goal**: Separate concerns.

**Actions**:
- **State Layer Separation**: Layers defined.
- **Add Selector Utilities**: Memoized.
- **Fix State Issues**: Corrections.

**Validation Gate**: Conflicts prevented.

**Files**: VariantStructureStudio.tsx, use-product-variant-actions.ts, variant-matrix.ts

### Phase 6 — Async Job Progress Streaming
**Goal**: Responsive progress via SSE.

**Actions**:
- **Implement SSE**: Events for streaming.
- **Connection Lifecycle Rules**: Handling.
- **Event Contract Definition**: Typed schemas.
- **Event Ordering Guarantees**: Sequence numbers, timestamps.
- **Resource Protection**: Limits, heartbeat.
- **Fallback Recovery**: Reconnect + REST fetch.
- **VariantJobEventBus**: Publisher emits events (renamed from JobProgressPublisher).

**Validation Gate**: Boundaries finalized.

**Files**: Frontend components, backend services

### Phase 7 — Testing Strategy (Business Guarantees)
**Goal**: Maintainable tests.

**Actions**:
- **Domain Logic Tests**: Transitions, rules.
- **Integration Tests**: DB, transactions.
- **UI Tests**: Flows.
- **E2E Tests**: Critical flows.
- **Fix Test Failures**: Corrections.

**Validation Gate**: 90%+ coverage.

**Files**: All test files

### Phase 8 — i18n Architecture
**Goal**: Proper i18n.

**Actions**:
- **Business Keys Independent**: Canonical IDs.
- **Presentation Localization**: Translations.
- **RTL Support**: Handling.
- **Storefront Consistency Rules**: DRAFT/LIVE.

**Validation Gate**: Tested.

**Files**: Frontend, variant-matrix.ts, schemas.py

### Phase 9 — Observability & Recovery
**Goal**: Monitoring.

**Actions**:
- **Structured Events**: Events defined.
- **Failure Classification**: Types.
- **Operational Observability Metrics**: Avg duration, queue wait, failed %, throughput, reconnect rate.
- **Audit Logging**: Tracking.
- **Job Recovery Rules**: Handling.
- **Error Recovery UI**: Messages.
- **API Contract Stability**: Versioning.

**Validation Gate**: Recovery tested.

**Files**: Backend services, frontend

### Phase 10 — Cleanup & Refactoring (LAST)
**Goal**: After stabilization.

**Actions**:
- **Remove Dead Code**: Consolidate.
- **Improve Messages**: Values included.
- **UX Polish**: Pagination.
- **Delayed Items**: Advanced features.

## SSE Infrastructure Requirements (AWS Deployment)
- **Nginx**: proxy_buffering off;
- **ALB**: Idle timeout
- **FastAPI**: Async
- **Workers**: Separate pools

## Progressive Scalability Plan
- **Stage 1**: FastAPI + SSE + in-memory
- **Stage 2**: FastAPI + Redis pub/sub + workers
- **Stage 3**: Distributed + event bus

## Execution Discipline
Implement core (domain, state machines, identity, SSE basics) first, then add quotas, backpressure, observability, recovery, scalability optimizations to avoid complexity explosion.

## Architecture Decision Records (ADRs)
Create ADRs for major decisions.

## Final Validation & Quality Gates
- **All Phase Gates**: Reviews required
- **Operational Readiness**: All tested
- **Build Gates**: Pass
- **Enterprise Standards**: Ready

This plan defines a production-ready commerce domain architecture with operational excellence.