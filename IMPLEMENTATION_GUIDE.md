# Implementation Guide for Code Agents

**Last Updated**: 2026-05-16  
**Status**: Critical Gaps Identified - Implement in Priority Order  
**Target**: Production-Ready Variants Feature

---

## 🎯 Quick Start for Code Agents

When starting implementation:
1. Read this guide completely
2. Review the priority matrix below
3. Start with **BLOCKER** items
4. Follow architectural patterns from existing code
5. All changes must pass tests, linting, and type checks
6. Update WHAT-DONE.md after each major phase

---

## 📊 Priority Matrix

### 🔴 BLOCKER (Stop Everything - Fix First)

| Task | Location | Effort | Impact |
|------|----------|--------|--------|
| **Implement i18n Framework** | PX-F/px, all error strings | 3-4 hours | CRITICAL - No multi-language support |
| **Fix Exception Handling in Worker** | PX-B/worker.py | 2 hours | CRITICAL - Swallows errors, no retry logic |
| **Add Error Classification** | PX-B/service.py | 2 hours | CRITICAL - Can't distinguish retryable vs permanent |

### 🟡 HIGH (This Sprint)

| Task | Location | Effort | Impact |
|------|----------|--------|--------|
| **Add Job Metrics & Observability** | PX-B/app/core + services | 4-5 hours | HIGH - Can't monitor production |
| **Optimize N+1 Queries** | PX-B/service.py | 3-4 hours | HIGH - Performance degradation at scale |
| **Rate Limit Job Endpoints** | PX-B/router.py + middleware | 2-3 hours | HIGH - DOS vulnerability |
| **Improve SSE Error Recovery UI** | PX-F/px/app/components | 2-3 hours | HIGH - Users can't recover from disconnects |

### 🟠 MEDIUM (Before Release)

| Task | Location | Effort | Impact |
|------|----------|--------|--------|
| Dead Code Audit | PX-B + PX-F | 2-3 hours | MEDIUM - Code maintainability |
| E2E Test Coverage | PX-B/tests + PX-F/test | 4-5 hours | MEDIUM - Integration gaps |
| Worker Connection Pool Tuning | PX-B/Makefile + config | 1-2 hours | MEDIUM - Resource efficiency |
| Frontend i18n Integration | PX-F/px/lib | 2-3 hours | MEDIUM - Consistency |

---

## 🔴 BLOCKER 1: i18n Framework Implementation

### What Needs to Do
Implement multi-language support throughout the application.

### Files to Modify/Create

#### Backend (PX-B)
- [ ] Create `app/core/i18n_keys.py` - Constants for all translatable strings
- [ ] Create `app/modules/catalog/i18n_keys.py` - Catalog-specific keys
- [ ] Update error messages to use keys instead of hardcoded strings
- [ ] Add translation loading to startup

#### Frontend (PX-F)
- [ ] Add `i18next` + `react-i18next` to `package.json`
- [ ] Create `lib/i18n/config.ts` - i18n configuration
- [ ] Create `public/locales/` directory structure
- [ ] Update all error messages, labels, buttons to use i18n

### Implementation Pattern

**Backend - Error Keys Pattern:**
```python
# app/modules/catalog/i18n_keys.py
VARIANT_ERRORS = {
    "JOB_ACTIVE": "catalog.variant.errors.job_active",
    "EXPLOSION_RISK": "catalog.variant.errors.explosion_risk",
    "STRUCTURE_STALE": "catalog.variant.errors.structure_stale",
    "INVALID_COMBINATION": "catalog.variant.errors.invalid_combination",
}

# Usage in service.py
if active_job is not None:
    raise CatalogVariantJobActiveError(
        message_key=VARIANT_ERRORS["JOB_ACTIVE"],
        details={"product_id": str(product_id)}
    )
```

**Frontend - Component Pattern:**
```typescript
// lib/i18n/config.ts
import i18n from 'i18next';
import { initReactI18next } from 'react-i18next';
import enTranslations from '@/public/locales/en/translations.json';
import esTranslations from '@/public/locales/es/translations.json';

i18n.use(initReactI18next).init({
  resources: {
    en: { translation: enTranslations },
    es: { translation: esTranslations },
  },
  lng: 'en',
  fallbackLng: 'en',
  ns: ['translation'],
  defaultNS: 'translation',
});

export default i18n;

// Usage in components
import { useTranslation } from 'react-i18next';

export function VariantJobError() {
  const { t } = useTranslation();
  return <div>{t('catalog.variant.errors.job_active')}</div>;
}
```

**Translation Files:**
```json
// public/locales/en/translations.json
{
  "catalog": {
    "variant": {
      "errors": {
        "job_active": "A variant generation job is already running for this product.",
        "explosion_risk": "Too many variants would be created. Reduce options or values.",
        "structure_stale": "Product structure changed. Please refresh and try again.",
        "invalid_combination": "Invalid option value combination selected."
      },
      "ui": {
        "generate_missing": "Generate Missing Variants",
        "rebuild_all": "Rebuild All Variants",
        "cancel_job": "Cancel Job",
        "progress": "{{created}} created, {{removed}} removed, {{preserved}} preserved"
      }
    }
  }
}
```

### Testing Requirements
- [ ] All error messages use i18n keys, not hardcoded strings
- [ ] Backend can handle translation key resolution
- [ ] Frontend loads correct locale based on store/user preference
- [ ] Missing translation keys logged but don't crash app
- [ ] `npm run i18n:check` passes (validates all keys)

### Validation Steps
```bash
# Backend
cd PX-B && python -m pytest tests/ -k "i18n or error" -v

# Frontend  
cd PX-F/px && npm run i18n:check && npm run test
```

---

## 🔴 BLOCKER 2: Fix Worker Exception Handling & Error Classification

### What Needs To Do
Implement proper error classification and retry logic in worker.

### Current Problem
```python
# PX-B/app/modules/catalog/worker.py - LINE 37
except Exception as exc:
    session.rollback()
    _mark_catalog_variant_job(session, job, "failed", error_message=str(exc))
```
**Issue**: All exceptions treated the same. No retry logic. No error classification.

### Solution: Error Classification System

#### Create `app/modules/catalog/job_errors.py`
```python
from enum import StrEnum

class JobErrorType(StrEnum):
    SYSTEM_ERROR = "system_error"          # Retry (transient: DB, network)
    VALIDATION_ERROR = "validation_error"  # Don't retry (bad data)
    CONFLICT_ERROR = "conflict_error"      # Don't retry (stale data)
    TIMEOUT_ERROR = "timeout_error"        # Retry once
    CANCELLED_ERROR = "cancelled_error"    # Terminal (user cancel)

class JobError(Exception):
    def __init__(self, error_type: JobErrorType, message: str, details: dict | None = None):
        self.error_type = error_type
        self.message = message
        self.details = details or {}
        super().__init__(message)

# Specific error types
class StructureStaleError(JobError):
    def __init__(self, message: str):
        super().__init__(JobErrorType.CONFLICT_ERROR, message)

class VariantExplosionError(JobError):
    def __init__(self, message: str):
        super().__init__(JobErrorType.VALIDATION_ERROR, message)

class DatabaseError(JobError):
    def __init__(self, message: str):
        super().__init__(JobErrorType.SYSTEM_ERROR, message)

# Retry classification
RETRYABLE_ERRORS = {JobErrorType.SYSTEM_ERROR, JobErrorType.TIMEOUT_ERROR}
NON_RETRYABLE_ERRORS = {JobErrorType.VALIDATION_ERROR, JobErrorType.CONFLICT_ERROR, JobErrorType.CANCELLED_ERROR}
```

#### Update `worker.py`
```python
# PX-B/app/modules/catalog/worker.py
import time
import logging
from app.modules.catalog.job_errors import JobError, JobErrorType, RETRYABLE_ERRORS

logger = logging.getLogger("variant-worker")

class VariantWorker:
    MAX_RETRIES = 3
    RETRY_DELAYS = [2, 5, 10]  # exponential backoff in seconds
    
    def run(self):
        logger.info("Variant Worker started")
        while not self.should_exit:
            try:
                job_id = self.fetch_next_job()
                if job_id:
                    self.execute_with_retry(job_id)
                else:
                    time.sleep(2)
            except Exception as e:
                logger.error(f"Worker loop error: {e}", exc_info=True)
                time.sleep(5)
    
    def execute_with_retry(self, job_id: UUID):
        """Execute job with retry logic for transient errors"""
        last_error = None
        
        for attempt in range(self.MAX_RETRIES + 1):
            try:
                execute_catalog_variant_job(job_id)
                return  # Success
            except JobError as e:
                last_error = e
                
                if e.error_type not in RETRYABLE_ERRORS:
                    logger.warning(f"Job {job_id} failed with non-retryable error: {e.error_type}")
                    break  # Don't retry
                
                if attempt < self.MAX_RETRIES:
                    delay = self.RETRY_DELAYS[attempt]
                    logger.info(f"Job {job_id} retry {attempt + 1}/{self.MAX_RETRIES} after {delay}s")
                    time.sleep(delay)
                else:
                    logger.error(f"Job {job_id} exhausted retries after {self.MAX_RETRIES} attempts")
                    
            except Exception as e:
                logger.error(f"Unexpected error in job {job_id}: {e}", exc_info=True)
                last_error = e
                break  # Don't retry unknown errors
        
        # Mark as failed if we got here
        if last_error:
            with Session(engine) as session:
                job = get_catalog_variant_job_by_id(session, job_id)
                if job:
                    _mark_catalog_variant_job(
                        session, job, "failed",
                        error_message=str(last_error)[:2000],
                        error_type=getattr(last_error, 'error_type', 'unknown')
                    )
```

#### Update `service.py` - Job Execution
```python
# PX-B/app/modules/catalog/service.py - Line 1813
def execute_catalog_variant_job(job_id: UUID) -> None:
    """Execute variant job with proper error handling"""
    with Session(engine) as session:
        job = get_catalog_variant_job_by_id(session, job_id)
        if not job:
            raise ValueError(f"Job {job_id} not found")
        
        try:
            _mark_catalog_variant_job(session, job, CatalogVariantJobStatus.RUNNING.value)
            
            # ... existing job execution logic ...
            
        except ProductStructureStaleHashError as e:
            raise StructureStaleError(f"Structure changed: {str(e)}")
        except JobCancelledError as e:
            _mark_catalog_variant_job(session, job, CatalogVariantJobStatus.CANCELLED.value)
            return
        except Exception as exc:
            logger.error(f"Job {job_id} failed: {exc}", exc_info=True)
            raise JobError(JobErrorType.SYSTEM_ERROR, str(exc)[:500])
```

### Testing Requirements
- [ ] Retryable errors retry with exponential backoff
- [ ] Non-retryable errors fail immediately
- [ ] Cancelled jobs don't retry
- [ ] Error details captured for logging
- [ ] Worker tests verify retry behavior

### Validation Steps
```bash
cd PX-B && python -m pytest tests/test_catalog_variant_ops.py -v -k "retry or error"
```

---

## 🔴 BLOCKER 3: Add Error Classification to Models

### What Needs To Do
Add `error_type` and `retry_count` fields to `CatalogVariantJob` model.

### Files to Modify
- `PX-B/app/modules/catalog/models.py` - Add fields to `CatalogVariantJob`
- `PX-B/alembic/versions/` - Create migration

### Changes to `CatalogVariantJob` Model
```python
class CatalogVariantJob(SQLModel, table=True):
    __tablename__ = "catalog_jobs"
    
    # ... existing fields ...
    
    error_type: str | None = Field(default=None, max_length=64, index=True)
    retry_count: int = Field(default=0)
    last_error_at: datetime | None = Field(default=None, index=True)
    
    # ... rest of model ...
```

### Migration
```bash
cd PX-B && alembic revision --autogenerate -m "add error classification to variant jobs"
```

---

## 🟡 HIGH PRIORITY 1: Add Observability Metrics

### What Needs To Do
Implement structured metrics for job performance, queue health, and error rates.

### Create `app/core/metrics.py`
```python
import time
import logging
from typing import Callable
from functools import wraps
from dataclasses import dataclass, field
from threading import Lock
from datetime import datetime, UTC
from statistics import mean, stdev

logger = logging.getLogger("metrics")

@dataclass
class JobMetrics:
    """Track job execution statistics"""
    job_id: str
    job_type: str
    start_time: datetime
    end_time: datetime | None = None
    status: str = "running"  # running, completed, failed, cancelled
    duration_ms: int = 0
    phase: str = "queued"
    error_type: str | None = None
    retry_count: int = 0
    
    def complete(self, status: str, error_type: str | None = None):
        self.end_time = datetime.now(UTC)
        self.duration_ms = int((self.end_time - self.start_time).total_seconds() * 1000)
        self.status = status
        self.error_type = error_type

class MetricsCollector:
    """Collect and aggregate job metrics"""
    
    def __init__(self):
        self._lock = Lock()
        self._jobs: dict[str, JobMetrics] = {}
        self._completed: list[JobMetrics] = []
        self._max_history = 1000
    
    def start_job(self, job_id: str, job_type: str) -> JobMetrics:
        """Mark job start"""
        metric = JobMetrics(
            job_id=job_id,
            job_type=job_type,
            start_time=datetime.now(UTC)
        )
        with self._lock:
            self._jobs[job_id] = metric
        logger.info(f"Job started", extra={"job_id": job_id, "job_type": job_type})
        return metric
    
    def complete_job(self, job_id: str, status: str, error_type: str | None = None):
        """Mark job complete"""
        with self._lock:
            metric = self._jobs.pop(job_id, None)
            if metric:
                metric.complete(status, error_type)
                self._completed.append(metric)
                if len(self._completed) > self._max_history:
                    self._completed.pop(0)
        logger.info(f"Job completed", extra={
            "job_id": job_id,
            "status": status,
            "duration_ms": metric.duration_ms if metric else 0
        })
    
    def get_stats(self) -> dict:
        """Get aggregated statistics"""
        with self._lock:
            active = len(self._jobs)
            completed = len(self._completed)
            failed = sum(1 for m in self._completed if m.status == "failed")
            cancelled = sum(1 for m in self._completed if m.status == "cancelled")
            
            durations = [m.duration_ms for m in self._completed if m.status == "completed"]
            avg_duration = mean(durations) if durations else 0
            p95_duration = sorted(durations)[int(len(durations) * 0.95)] if len(durations) > 20 else 0
            
            return {
                "active_jobs": active,
                "total_completed": completed,
                "failed_count": failed,
                "cancelled_count": cancelled,
                "success_rate": (completed - failed) / completed * 100 if completed > 0 else 0,
                "avg_duration_ms": avg_duration,
                "p95_duration_ms": p95_duration,
            }

# Global instance
metrics_collector = MetricsCollector()

def track_job_execution(job_type: str):
    """Decorator to track job execution metrics"""
    def decorator(func: Callable):
        @wraps(func)
        def wrapper(job_id: UUID, *args, **kwargs):
            metric = metrics_collector.start_job(str(job_id), job_type)
            try:
                result = func(job_id, *args, **kwargs)
                metrics_collector.complete_job(str(job_id), "completed")
                return result
            except Exception as e:
                error_type = getattr(e, 'error_type', type(e).__name__)
                metrics_collector.complete_job(str(job_id), "failed", error_type)
                raise
        return wrapper
    return decorator
```

### Update `worker.py` to use metrics
```python
from app.core.metrics import metrics_collector

def execute_with_retry(self, job_id: UUID):
    """Execute job with retry logic"""
    metric = metrics_collector.start_job(str(job_id), "variant_generation")
    # ... existing retry logic ...
```

### Create `app/modules/catalog/routes/metrics.py`
```python
from fastapi import APIRouter, Depends
from app.core.metrics import metrics_collector

router = APIRouter(prefix="/admin/metrics", tags=["metrics"])

@router.get("/variant-jobs")
def get_variant_job_metrics():
    """Get job queue and performance metrics"""
    return metrics_collector.get_stats()
```

### Testing
- [ ] Metrics collected for all job types
- [ ] Stats API returns correct values
- [ ] Performance tracked per phase
- [ ] Error rates tracked

---

## 🟡 HIGH PRIORITY 2: Optimize N+1 Queries

### Problem Areas

**Location 1**: `service.py` - `_list_sorted_product_options_with_values()`
```python
# Current (N+1):
for option in list_product_options(session, product_id=product.id):
    for value in list_product_option_values(session, option_id=option.id):  # N queries
        # ...

# Should use SQLAlchemy relationship loading
```

**Location 2**: `service.py` - Variant rendering
```python
# Current (N+1):
for variant in list_product_variants(...):
    values = list_variant_value_links(session, variant_id=variant.id)  # N queries

# Should fetch all at once
```

### Solution Pattern
```python
# Use selectinload or joinedload
from sqlalchemy.orm import selectinload, joinedload

def get_options_with_values_optimized(session, product_id: UUID):
    """Fetch options and values in 1 query"""
    options = session.exec(
        select(ProductOption)
        .where(ProductOption.product_id == product_id)
        .options(selectinload(ProductOption.values))
        .order_by(ProductOption.position)
    ).unique().all()
    return options
```

### Files to Optimize
- [ ] `service.py` - Line ~1900 (options with values)
- [ ] `service.py` - Line ~2100 (variant rendering)
- [ ] `service.py` - Line ~2400 (snapshot creation)

### Testing
```bash
cd PX-B && pytest tests/ -v -k "performance" --durations=10
```

---

## 🟡 HIGH PRIORITY 3: Rate Limiting for Job Endpoints

### Create `app/middleware/rate_limit.py`
```python
from fastapi import Request, HTTPException, status
from functools import lru_cache
import time
from uuid import UUID

class RateLimiter:
    """Simple in-memory rate limiter"""
    
    def __init__(self):
        self._requests: dict[str, list[float]] = {}
    
    def is_allowed(self, key: str, max_requests: int, window_seconds: int) -> bool:
        """Check if request is allowed"""
        now = time.time()
        cutoff = now - window_seconds
        
        if key not in self._requests:
            self._requests[key] = []
        
        # Cleanup old requests
        self._requests[key] = [t for t in self._requests[key] if t > cutoff]
        
        if len(self._requests[key]) >= max_requests:
            return False
        
        self._requests[key].append(now)
        return True

# Per-endpoint limiters
job_creation_limiter = RateLimiter()

async def rate_limit_job_creation(user_id: UUID, store_id: UUID) -> bool:
    """Limit: 10 jobs per user per minute, 50 per store per minute"""
    user_key = f"user_jobs:{user_id}"
    store_key = f"store_jobs:{store_id}"
    
    user_allowed = job_creation_limiter.is_allowed(user_key, max_requests=10, window_seconds=60)
    store_allowed = job_creation_limiter.is_allowed(store_key, max_requests=50, window_seconds=60)
    
    return user_allowed and store_allowed
```

### Update Router
```python
# router.py - variant job endpoints

@router.post("/admin/products/{product_id}/variant-jobs/generate-missing")
async def create_generate_missing_variant_job_admin(
    current_user: ActiveUser = Depends(),
):
    if not await rate_limit_job_creation(current_user.id, product.store_id):
        raise HTTPException(
            status_code=status.HTTP_429_TOO_MANY_REQUESTS,
            detail="Too many job requests. Please wait before submitting another."
        )
    # ... existing logic ...
```

---

## 🟡 HIGH PRIORITY 4: SSE Error Recovery UI

### Current State
- EventSource reconnects but no UI feedback
- No user-facing error messages
- Silent failures

### Needed Improvements
```typescript
// lib/catalog/variant-job-events.ts
export function useVariantJobEventStream(jobId: UUID) {
  const [state, setState] = useState<'connecting' | 'connected' | 'error' | 'reconnecting'>('connecting');
  const [lastError, setLastError] = useState<string | null>(null);
  const [reconnectAttempts, setReconnectAttempts] = useState(0);
  
  useEffect(() => {
    let reconnectTimeout: NodeJS.Timeout;
    
    const connect = () => {
      setState('connecting');
      const source = new EventSource(url, { withCredentials: true });
      
      source.addEventListener('error', () => {
        setLastError('Connection lost. Reconnecting...');
        setState('reconnecting');
        source.close();
        
        if (reconnectAttempts < 5) {
          const delay = Math.min(1000 * Math.pow(2, reconnectAttempts), 30000);
          reconnectTimeout = setTimeout(() => {
            setReconnectAttempts(a => a + 1);
            connect();
          }, delay);
        } else {
          setLastError('Connection failed. Please refresh the page.');
          setState('error');
        }
      });
      
      source.addEventListener('open', () => {
        setState('connected');
        setLastError(null);
        setReconnectAttempts(0);
      });
    };
    
    connect();
    
    return () => {
      clearTimeout(reconnectTimeout);
      source?.close();
    };
  }, [jobId]);
  
  return { state, lastError, reconnectAttempts };
}
```

---

## 📋 Testing Checklist

### Backend Tests Required
- [ ] All error types throw correct `JobError` subclasses
- [ ] Worker retries transient errors, not permanent ones
- [ ] Metrics collected and aggregated correctly
- [ ] N+1 query fixes verified with query logging
- [ ] Rate limiters work correctly
- [ ] i18n keys all present and used

### Frontend Tests Required
- [ ] i18n provider initializes correctly
- [ ] All error messages use translation keys
- [ ] SSE reconnection shows UI feedback
- [ ] Rate limit 429 handled gracefully

### All Tests Must Pass
```bash
# Backend
cd PX-B && python -m pytest tests/ -v --cov=app --cov-report=term-missing

# Frontend
cd PX-F/px && npm test -- --coverage
```

### Linting & Type Checking
```bash
# Backend
cd PX-B && ruff check . && mypy app/

# Frontend
cd PX-F/px && npm run lint && npm run typecheck
```

---

## ✅ Completion Criteria

### Before Marking Complete
- [ ] All BLOCKER items fixed
- [ ] All HIGH items fixed
- [ ] 100% test pass rate
- [ ] 0 lint errors
- [ ] 0 type errors
- [ ] Code reviewed for consistency
- [ ] WHAT-DONE.md updated with details
- [ ] Performance benchmarks show improvements

### Success Metrics
- Job error rate classified correctly
- Worker retry logic functioning
- i18n working for all languages
- Observability metrics accessible
- Queries optimized (N+1 eliminated)
- Rate limiting protecting endpoints
- All tests passing

---

## 📞 Quick Reference

### Key Files
- Models: `PX-B/app/modules/catalog/models.py`
- Service: `PX-B/app/modules/catalog/service.py`
- Worker: `PX-B/app/modules/catalog/worker.py`
- Router: `PX-B/app/modules/catalog/router.py`
- Frontend: `PX-F/px/app/components/VariantStructureStudio.tsx`

### Important Functions
- Job execution: `execute_catalog_variant_job()`
- Error handling: `_mark_catalog_variant_job()`
- Structure guard: `assert_structure_guard()`
- Snapshot creation: `capture_catalog_product_structure_snapshot()`

### Testing Commands
```bash
# Quick backend test
cd PX-B && python -m pytest tests/test_catalog_variant_ops.py -v

# Full test suite
cd PX-B && python -m pytest tests/ -v

# Frontend
cd PX-F/px && npm test
```

---

**Last Updated**: 2026-05-16  
**Next Review**: After all BLOCKER items completed
