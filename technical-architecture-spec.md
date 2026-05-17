# Technical Architecture Specification

Status note, 2026-05-17:
This remains an architecture reference. For current completion status and next work, read `AGENT_HANDOFF.md` and `WHAT-DONE.md` first.

## Purpose
This document provides implementation-focused guidance for the variants feature, detailing the technical architecture, contracts, and patterns.

## SSE Architecture
- **Purpose**: Stream job lifecycle progress to clients
- **Implementation**: Server-Sent Events (SSE) over HTTP
- **Connection Management**:
  - Reconnect on failure with exponential backoff
  - Heartbeat every 15-30 seconds
  - Idle timeout: 5 minutes
  - Max connections per user: 3, per store: 10
- **Event Ordering**: Monotonic sequence numbers per job
- **Event Idempotency**: Events are replay-safe and idempotent by sequence number
- **Event Versioning**: Include eventVersion in payloads for schema evolution
- **Event Schema**:
  - progress: {jobId, sequence, percent, message}
  - state: {jobId, sequence, state, timestamp}
  - completed: {jobId, sequence, result}
  - failed: {jobId, sequence, errorType, message}
  - cancelled: {jobId, sequence, reason}

## Worker Boundaries
- **Separation**: Workers run in separate process pools from API
- **Communication**: Queue-based (initially in-memory, scale to Redis)
- **Isolation**: Heavy generation never blocks API or SSE
- **Prioritization**: High (publish/sync), Medium (generation), Low (cleanup/rebuild)
- **Idempotency**: Workers handle duplicate executions safely
- **Cancellation**: Cooperative, checkpoint-based, cleanup-aware

## Event Flow
REST API → Enqueue Job → Worker Processes → VariantJobEventBus Publishes → SSE Streams → Client Receives

## DB Contracts
- **Variant Identity**: combination_key as hashed canonical string
- **Indexes**: (product_id, combination_key), (store_id, status)
- **Constraints**: Unique (product_id, combination_key)
- **Versioning**: structure_version, snapshot_version, published_version columns

## Snapshot Flow
1. Job starts: Capture current structure as snapshot
2. Execute generation against snapshot
3. On success: Swap published snapshot atomically
4. Store snapshot metadata for rollback/debugging

## Retry Semantics
- **API Level**: Idempotency keys for 24 hours
- **Worker Level**: At-least-once with deduplication
- **Jitter**: Randomized jitter on exponential backoff to prevent retry storms
- **Classification**:
  - SYSTEM_FAILED: Retry up to 3 times
  - NETWORK_FAILURE: Retry with backoff
  - VALIDATION_FAILED: No retry
  - CONFLICT_FAILED: No retry
  - TIMEOUT: Retry once

## API Contracts
- **Versioning**: Semantic versioning, backward compatible
- **Stability**: Event schemas evolve with optional fields
- **Security**: Signed SSE streams, auth validation per event
- **Consistency**: Read-after-write guarantees for structure mutations

## State Ownership
- **Structure**: DB (authoritative)
- **Job Progress**: Worker memory/pubsub
- **Storefront**: Published snapshots (never read from draft structures)
- **UI Draft**: Client state

## Async Boundaries
- **Synchronous**: Structure edits, validations
- **Asynchronous**: Generation jobs, sync operations
- **Transactional**: Publication swaps
- **Forbidden Behaviors**:
  - No generation inside request lifecycle
  - No DB polling in SSE loops
  - No direct UI mutation bypassing lifecycle APIs
  - No direct variant mutation bypassing capability guards

## Scalability Stages
- **Stage 1**: In-memory queues, FastAPI workers
- **Stage 2**: Redis pub/sub, dedicated workers
- **Stage 3**: Distributed event bus, containerized workers

## Multi-Region Future Compatibility
- Event ordering guarantees are scoped per job and not globally distributed

## Security
- **Tenant Isolation**: Store-scoped queries
- **Auth**: JWT validation on SSE connections
- **Quotas**: Enforced at API layer
- **Audit**: All mutations logged

## Observability
- **Metrics**: Job duration, failure rates, throughput
- **Events**: Structured logging for monitoring
- **Recovery**: Snapshot rollback, job restart

This spec guides all technical implementations.
