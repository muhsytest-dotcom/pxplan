# Error Classification & Worker Retry Logic Implementation

**Task**: Implement proper error classification and retry strategies  
**Priority**: 🔴 BLOCKER  
**Estimated Effort**: 3-4 hours  
**Files**: worker.py, service.py, models.py (+ migration)

---

## Overview

Currently all exceptions in the worker are caught and failed uniformly. This guide implements proper error classification enabling smart retry logic.

---

## Phase 1: Create Error Classification System

### Step 1: Create `PX-B/app/modules/catalog/job_errors.py`

```python
"""
Error classification system for variant jobs.

All variant job errors must inherit from JobError and specify:
  - error_type: Whether to retry
  - message: Human-readable error description
  - details: Context for logging/UI
"""

from enum import StrEnum
from typing import Any, Optional

class JobErrorType(StrEnum):
    """Classification of errors for retry logic"""
    
    # Transient system errors - RETRY with exponential backoff
    SYSTEM_ERROR = "system_error"              # DB connection, network, disk
    NETWORK_ERROR = "network_error"            # Network timeouts
    RESOURCE_EXHAUSTED = "resource_exhausted"  # Out of memory, disk full
    
    # Transient conflict - RETRY with longer backoff
    TIMEOUT_ERROR = "timeout_error"            # Job exceeded time limit
    LOCK_TIMEOUT = "lock_timeout"              # DB lock acquisition timeout
    
    # Permanent errors - DON'T RETRY
    VALIDATION_ERROR = "validation_error"      # Invalid input, bad structure
    CONFLICT_ERROR = "conflict_error"          # Stale version, race condition
    AUTHORIZATION_ERROR = "authorization_error" # Permission denied
    NOT_FOUND_ERROR = "not_found_error"        # Resource doesn't exist
    
    # User actions - DON'T RETRY
    CANCELLED_ERROR = "cancelled_error"        # User cancelled the job
    
    # Unknown - LOG AND DON'T RETRY
    UNKNOWN_ERROR = "unknown_error"            # Unexpected error

# Retry classification
RETRYABLE_ERRORS = {
    JobErrorType.SYSTEM_ERROR,
    JobErrorType.NETWORK_ERROR,
    JobErrorType.RESOURCE_EXHAUSTED,
    JobErrorType.TIMEOUT_ERROR,
    JobErrorType.LOCK_TIMEOUT,
}

NON_RETRYABLE_ERRORS = {
    JobErrorType.VALIDATION_ERROR,
    JobErrorType.CONFLICT_ERROR,
    JobErrorType.AUTHORIZATION_ERROR,
    JobErrorType.NOT_FOUND_ERROR,
    JobErrorType.CANCELLED_ERROR,
    JobErrorType.UNKNOWN_ERROR,
}

class JobError(Exception):
    """Base class for all variant job errors"""
    
    def __init__(
        self,
        error_type: JobErrorType,
        message: str,
        i18n_key: Optional[str] = None,
        details: Optional[dict[str, Any]] = None,
        cause: Optional[Exception] = None,
    ):
        self.error_type = error_type
        self.message = message
        self.i18n_key = i18n_key
        self.details = details or {}
        self.cause = cause
        
        # Build full message
        full_message = f"[{error_type}] {message}"
        if details:
            full_message += f" | {details}"
        super().__init__(full_message)
    
    def is_retryable(self) -> bool:
        """Check if this error type should be retried"""
        return self.error_type in RETRYABLE_ERRORS

# ============================================================
# Specific Error Classes
# ============================================================

class StructureStaleError(JobError):
    """Product structure changed during job execution"""
    def __init__(self, message: str = "Product structure was modified"):
        super().__init__(
            error_type=JobErrorType.CONFLICT_ERROR,
            message=message,
            i18n_key="catalog.variant.errors.structure_stale_version",
        )

class VariantExplosionError(JobError):
    """Too many variants would be created"""
    def __init__(self, message: str, projected_count: int, limit: int):
        super().__init__(
            error_type=JobErrorType.VALIDATION_ERROR,
            message=message,
            i18n_key="catalog.variant.errors.explosion_risk",
            details={"projected": projected_count, "limit": limit},
        )

class DatabaseError(JobError):
    """Database connection or transaction error"""
    def __init__(self, message: str, cause: Exception):
        super().__init__(
            error_type=JobErrorType.SYSTEM_ERROR,
            message=message,
            details={"db_error": type(cause).__name__},
            cause=cause,
        )

class JobCancelledError(JobError):
    """Job was cancelled by user or system"""
    def __init__(self, reason: str = "Job cancelled"):
        super().__init__(
            error_type=JobErrorType.CANCELLED_ERROR,
            message=reason,
            i18n_key="catalog.variant.errors.job_cancelled",
        )

class JobTimeoutError(JobError):
    """Job exceeded execution time"""
    def __init__(self, duration_seconds: int):
        super().__init__(
            error_type=JobErrorType.TIMEOUT_ERROR,
            message=f"Job timeout after {duration_seconds}s",
            i18n_key="catalog.variant.errors.job_timeout",
            details={"duration_seconds": duration_seconds},
        )

class ResourceExhaustedError(JobError):
    """System resources (memory, disk) exhausted"""
    def __init__(self, message: str, resource: str):
        super().__init__(
            error_type=JobErrorType.RESOURCE_EXHAUSTED,
            message=message,
            details={"resource": resource},
        )

class InvalidProductError(JobError):
    """Product structure invalid"""
    def __init__(self, message: str, product_id: str):
        super().__init__(
            error_type=JobErrorType.VALIDATION_ERROR,
            message=message,
            i18n_key="catalog.variant.errors.structure_invalid",
            details={"product_id": product_id},
        )

# Error wrapping helper
def wrap_error(exc: Exception, context: str = "") -> JobError:
    """Convert unknown exceptions to JobError"""
    if isinstance(exc, JobError):
        return exc
    
    exc_type = type(exc).__name__
    
    # Map specific exception types to JobError types
    if isinstance(exc, (ConnectionError, TimeoutError)):
        return JobError(
            JobErrorType.NETWORK_ERROR,
            f"{context}: {str(exc)}",
            cause=exc,
        )
    
    if "database" in str(exc).lower() or "db" in str(exc).lower():
        return DatabaseError(f"{context}: {str(exc)}", exc)
    
    if "memory" in str(exc).lower():
        return ResourceExhaustedError(f"{context}: {str(exc)}", "memory")
    
    # Default: unknown error
    return JobError(
        JobErrorType.UNKNOWN_ERROR,
        f"{context}: {str(exc)}",
        details={"exception_type": exc_type},
        cause=exc,
    )
```

### Step 2: Add Fields to `CatalogVariantJob` Model

File: `PX-B/app/modules/catalog/models.py` - Add to `CatalogVariantJob` class:

```python
class CatalogVariantJob(SQLModel, table=True):
    __tablename__ = "catalog_jobs"
    
    # ... existing fields ...
    
    # Error tracking (NEW FIELDS)
    error_type: str | None = Field(
        default=None,
        max_length=64,
        index=True,
        description="Classification of error for retry decisions"
    )
    retry_count: int = Field(
        default=0,
        description="Number of times this job has been retried"
    )
    last_error_at: datetime | None = Field(
        default=None,
        index=True,
        description="Timestamp of last error (for retry backoff)"
    )
    last_error_message: str | None = Field(
        default=None,
        max_length=500,
        description="Most recent error message"
    )
    
    # ... rest of model ...
```

### Step 3: Create Alembic Migration

```bash
cd PX-B
alembic revision --autogenerate -m "add error classification fields to variant jobs"
```

This should generate a migration that adds the new columns. Verify the migration in `PX-B/alembic/versions/`.

---

## Phase 2: Update Service Layer

### Step 1: Update `_mark_catalog_variant_job()` Function

File: `PX-B/app/modules/catalog/service.py` - Find and update this function (~line 1961):

```python
def _mark_catalog_variant_job(
    session: Session,
    job: CatalogVariantJob,
    status: str,
    phase: str | None = None,
    error_message: str | None = None,
    error_type: str | None = None,  # NEW PARAMETER
    retry_count: int | None = None,  # NEW PARAMETER
) -> None:
    """Mark job status and optional error details"""
    job.status = status
    
    if phase is not None:
        job.phase = phase
    
    if error_message is not None:
        job.error_message = error_message[:2000]  # Truncate
        job.last_error_at = utcnow()
    
    if error_type is not None:
        job.error_type = error_type
    
    if retry_count is not None:
        job.retry_count = retry_count
    
    if status == CatalogVariantJobStatus.COMPLETED.value:
        job.finished_at = utcnow()
    elif status == CatalogVariantJobStatus.FAILED.value:
        job.finished_at = utcnow()
    elif status == CatalogVariantJobStatus.CANCELLED.value:
        job.finished_at = utcnow()
    elif status == CatalogVariantJobStatus.TIMEOUT.value:
        job.finished_at = utcnow()
    
    save_catalog_variant_job(session, job, commit=False)
    
    # Also create durable event record
    _transition_variant_job(session, job)
```

### Step 2: Update Job Execution to Raise `JobError`

File: `PX-B/app/modules/catalog/service.py` - Update `execute_catalog_variant_job()` (~line 1813):

```python
from app.modules.catalog.job_errors import (
    JobError, JobErrorType, StructureStaleError, 
    InvalidProductError, wrap_error
)

def execute_catalog_variant_job(job_id: UUID) -> None:
    """Execute variant generation job with proper error handling"""
    with Session(engine) as session:
        job = get_catalog_variant_job_by_id(session, job_id)
        if not job:
            raise ValueError(f"Job {job_id} not found")
        
        try:
            # Mark as running
            _mark_catalog_variant_job(session, job, CatalogVariantJobStatus.RUNNING.value)
            session.commit()
            
            # Fetch product
            product = get_product_by_id(session, job.product_id)
            if not product:
                raise InvalidProductError(
                    "Product not found",
                    str(job.product_id)
                )
            
            # Run job phases
            _run_variant_job_phases(session, job, product)
            
            # Mark as completed
            _mark_catalog_variant_job(
                session, job, CatalogVariantJobStatus.COMPLETED.value
            )
            session.commit()
            
        except JobError as e:
            logger.warning(
                f"Job {job_id} failed with {e.error_type}: {e.message}",
                extra={"job_id": str(job_id), "error_type": e.error_type}
            )
            _mark_catalog_variant_job(
                session, job,
                status=CatalogVariantJobStatus.FAILED.value,
                error_message=e.message[:500],
                error_type=e.error_type,
            )
            session.commit()
            raise  # Re-raise for worker to handle retry
            
        except Exception as exc:
            logger.error(
                f"Unexpected error in job {job_id}: {exc}",
                exc_info=True
            )
            job_error = wrap_error(exc, "Job execution")
            _mark_catalog_variant_job(
                session, job,
                status=CatalogVariantJobStatus.FAILED.value,
                error_message=job_error.message[:500],
                error_type=job_error.error_type,
            )
            session.commit()
            raise job_error

def _run_variant_job_phases(
    session: Session,
    job: CatalogVariantJob,
    product: Product
) -> None:
    """Execute job phases with error handling"""
    try:
        # Phase 1: Validate
        _mark_catalog_variant_job(
            session, job,
            phase=CatalogVariantJobPhase.VALIDATING.value
        )
        _validate_job_preconditions(session, job, product)
        
        # Phase 2: Generate combinations
        _mark_catalog_variant_job(
            session, job,
            phase=CatalogVariantJobPhase.COMPUTING_COMBINATIONS.value
        )
        combinations = _compute_variant_combinations(session, product)
        
        # Phase 3: Create variants
        _mark_catalog_variant_job(
            session, job,
            phase=CatalogVariantJobPhase.CREATING_VARIANTS.value
        )
        created_count = _create_variants_for_combinations(
            session, job, product, combinations
        )
        
        # Update counts
        job.created_count = created_count
        
        # ... rest of phases ...
        
        session.flush()
        
    except ProductStructureStaleVersionError:
        raise StructureStaleError("Product version changed during job")
    except ProductStructureStaleHashError:
        raise StructureStaleError("Product structure changed during job")
    except Exception as exc:
        raise wrap_error(exc, "Job phase execution")
```

---

## Phase 3: Update Worker with Retry Logic

### Step 1: Update `PX-B/app/modules/catalog/worker.py`

```python
"""
Variant job worker with intelligent retry logic.
"""

import time
import logging
import signal
from uuid import UUID
from sqlmodel import Session, select

from app.db.session import engine
from app.modules.catalog.models import CatalogVariantJob, CatalogVariantJobStatus
from app.modules.catalog.service import execute_catalog_variant_job
from app.modules.catalog.repository import get_catalog_variant_job_by_id
from app.modules.catalog.job_errors import (
    JobError, RETRYABLE_ERRORS, NON_RETRYABLE_ERRORS
)

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger("variant-worker")

class VariantWorker:
    """Background worker for async variant jobs"""
    
    # Retry configuration
    MAX_RETRIES = 3
    RETRY_DELAYS = [2, 5, 10]  # seconds: 1st=2s, 2nd=5s, 3rd=10s
    JOB_POLLING_INTERVAL = 2   # seconds between job polls
    
    def __init__(self):
        self.should_exit = False
        signal.signal(signal.SIGINT, self._handle_signal)
        signal.signal(signal.SIGTERM, self._handle_signal)
    
    def _handle_signal(self, signum, frame):
        """Graceful shutdown on SIGTERM/SIGINT"""
        logger.info("Termination signal received. Finishing current job...")
        self.should_exit = True
    
    def run(self):
        """Main worker loop"""
        logger.info("Variant Worker started. Polling for jobs...")
        
        while not self.should_exit:
            try:
                job_id = self._fetch_next_job()
                if job_id:
                    logger.info(f"Picked up job {job_id}")
                    self._execute_with_retry(job_id)
                    logger.info(f"Finished job {job_id}")
                else:
                    time.sleep(self.JOB_POLLING_INTERVAL)
            
            except Exception as e:
                logger.error(f"Worker loop error: {e}", exc_info=True)
                time.sleep(5)
    
    def _fetch_next_job(self) -> UUID | None:
        """Atomically claim next QUEUED job"""
        try:
            with Session(engine) as session:
                # Find first QUEUED job
                statement = select(CatalogVariantJob).where(
                    CatalogVariantJob.status == CatalogVariantJobStatus.QUEUED.value
                ).order_by(CatalogVariantJob.created_at)
                
                job = session.exec(statement).first()
                if not job:
                    return None
                
                # Try to claim atomically
                job.status = CatalogVariantJobStatus.RUNNING.value
                session.add(job)
                session.commit()
                
                return job.id
        
        except Exception as e:
            logger.error(f"Error fetching job: {e}", exc_info=True)
            return None
    
    def _execute_with_retry(self, job_id: UUID) -> None:
        """Execute job with exponential backoff retry logic"""
        
        for attempt in range(self.MAX_RETRIES + 1):
            try:
                execute_catalog_variant_job(job_id)
                logger.info(f"Job {job_id} succeeded on attempt {attempt + 1}")
                return
            
            except JobError as e:
                logger.warning(
                    f"Job {job_id} attempt {attempt + 1}: {e.error_type}",
                    extra={
                        "job_id": str(job_id),
                        "error_type": e.error_type,
                        "is_retryable": e.is_retryable(),
                    }
                )
                
                # Non-retryable: give up immediately
                if not e.is_retryable():
                    logger.info(f"Job {job_id} failed with non-retryable error: {e.error_type}")
                    self._mark_job_failed(job_id, e)
                    return
                
                # Retryable but exhausted retries
                if attempt >= self.MAX_RETRIES:
                    logger.error(f"Job {job_id} exhausted retries ({self.MAX_RETRIES})")
                    self._mark_job_failed(job_id, e)
                    return
                
                # Retryable: wait and retry
                delay = self.RETRY_DELAYS[attempt]
                logger.info(
                    f"Job {job_id} retry {attempt + 1}/{self.MAX_RETRIES} "
                    f"in {delay}s (error: {e.error_type})"
                )
                time.sleep(delay)
            
            except Exception as exc:
                logger.error(
                    f"Unexpected error in job {job_id}: {exc}",
                    exc_info=True
                )
                self._mark_job_failed_unexpected(job_id, exc)
                return
    
    def _mark_job_failed(self, job_id: UUID, error: JobError) -> None:
        """Mark job as failed with error details"""
        try:
            with Session(engine) as session:
                job = get_catalog_variant_job_by_id(session, job_id)
                if job:
                    job.status = CatalogVariantJobStatus.FAILED.value
                    job.error_type = error.error_type
                    job.error_message = error.message[:500]
                    job.finished_at = time.time()  # Use UTC timestamp
                    session.add(job)
                    session.commit()
        except Exception as e:
            logger.error(f"Error marking job {job_id} as failed: {e}", exc_info=True)
    
    def _mark_job_failed_unexpected(self, job_id: UUID, exc: Exception) -> None:
        """Mark job as failed due to unexpected error"""
        try:
            with Session(engine) as session:
                job = get_catalog_variant_job_by_id(session, job_id)
                if job:
                    job.status = CatalogVariantJobStatus.FAILED.value
                    job.error_type = "unknown_error"
                    job.error_message = f"Unexpected error: {str(exc)[:400]}"
                    job.finished_at = time.time()
                    session.add(job)
                    session.commit()
        except Exception as e:
            logger.error(f"Error marking job {job_id} as failed: {e}", exc_info=True)

if __name__ == "__main__":
    worker = VariantWorker()
    worker.run()
```

### Step 2: Update Makefile Targets

File: `PX-B/Makefile` - Ensure worker targets exist:

```makefile
.PHONY: run-worker
run-worker:
	python app/modules/catalog/worker.py

.PHONY: dev-worker
dev-worker:
	watch -n 1 "python app/modules/catalog/worker.py" --clear
```

---

## Testing

### Backend Tests

Create `PX-B/tests/test_job_error_classification.py`:

```python
import pytest
from app.modules.catalog.job_errors import (
    JobError, JobErrorType, StructureStaleError,
    VariantExplosionError, RETRYABLE_ERRORS, NON_RETRYABLE_ERRORS,
)

class TestErrorClassification:
    """Test error types and retry logic"""
    
    def test_retryable_errors(self):
        """System/network errors should be retryable"""
        error = JobError(JobErrorType.SYSTEM_ERROR, "DB connection failed")
        assert error.is_retryable()
        assert error.error_type in RETRYABLE_ERRORS
    
    def test_non_retryable_errors(self):
        """Validation errors should not be retried"""
        error = JobError(JobErrorType.VALIDATION_ERROR, "Invalid input")
        assert not error.is_retryable()
        assert error.error_type in NON_RETRYABLE_ERRORS
    
    def test_structure_stale_error_not_retryable(self):
        """Stale structure is a conflict, not retryable"""
        error = StructureStaleError()
        assert error.error_type == JobErrorType.CONFLICT_ERROR
        assert not error.is_retryable()
    
    def test_explosion_error_not_retryable(self):
        """Explosion risk is validation error, not retryable"""
        error = VariantExplosionError("Too many", 10000, 5000)
        assert error.error_type == JobErrorType.VALIDATION_ERROR
        assert not error.is_retryable()

class TestWorkerRetryLogic:
    """Test worker retry behavior"""
    
    def test_retryable_error_retries(self):
        """Worker retries transient errors"""
        # Mock job with retryable error
        # Verify it's retried up to MAX_RETRIES times
        pass
    
    def test_non_retryable_error_fails_immediately(self):
        """Worker fails immediately on non-retryable errors"""
        # Mock job with validation error
        # Verify it's marked failed after 1 attempt
        pass
    
    def test_exponential_backoff(self):
        """Worker uses exponential backoff: 2s, 5s, 10s"""
        # Verify delays between retries
        pass

# Run tests
# pytest tests/test_job_error_classification.py -v
```

### Run Tests

```bash
cd PX-B

# Test error classification
python -m pytest tests/test_job_error_classification.py -v

# Test full job flow
python -m pytest tests/test_catalog_variant_ops.py -v

# All tests
python -m pytest tests/ -v
```

---

## Validation Checklist

- [ ] `JobError` base class with subtypes created
- [ ] `RETRYABLE_ERRORS` and `NON_RETRYABLE_ERRORS` defined
- [ ] Model migration created and applied
- [ ] `_mark_catalog_variant_job()` updated with error fields
- [ ] `execute_catalog_variant_job()` raises `JobError` on failures
- [ ] Worker implements exponential backoff retry
- [ ] Worker doesn't retry non-retryable errors
- [ ] All tests pass
- [ ] Worker logs include error classification

---

## Summary

This implementation ensures:
- ✅ Transient errors (DB, network) are retried with backoff
- ✅ Permanent errors (validation, conflicts) fail immediately
- ✅ User cancellations don't retry
- ✅ All errors tracked with classification for monitoring
- ✅ Clear audit trail for debugging

---

**Status**: Ready for implementation  
**Owner**: Code Agent  
**Due**: As part of BLOCKER items
