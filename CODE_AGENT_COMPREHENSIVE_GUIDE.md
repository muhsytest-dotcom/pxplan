# CODE AGENT COMPREHENSIVE GUIDE
## Production-Ready Implementation with Testing Standards

**Last Updated**: 2026-05-17  
**Status**: Critical Path Implementation  
**Target**: Production-Grade Variants Feature (100% Complete)

---

## 🎯 WHAT CODE AGENTS MUST UNDERSTAND

When reading this folder's documents, understand that:

1. **The system is 65-70% complete** with 3 critical blocking issues
2. **Planning is excellent** but execution gaps exist in error handling, i18n, and testing
3. **Production-ready means**: Passes tests, linting, type checks, and handles real failures
4. **This is NOT a suggestion** - these are architectural requirements from the domain contract
5. **You must implement in priority order** - dependencies exist between changes

---

## 📊 CRITICAL ASSESSMENT TABLE

| Issue | Status | Effort | Impact | Must Fix |
|-------|--------|--------|--------|----------|
| Backend Error Classification | 0% | 3-4h | CRITICAL - No retry logic | ✅ YES |
| Worker Retry Logic | 0% | 2-3h | CRITICAL - Permanent failures | ✅ YES |
| Backend i18n Integration | 0% | 3-4h | HIGH - Non-English errors fail | ✅ YES |
| Error Classification Tests | 0% | 2h | HIGH - No error behavior verification | ✅ YES |
| i18n Integration Tests | 0% | 2h | HIGH - Translations untested | ✅ YES |
| Dead Code Audit | Not Done | 2-3h | MEDIUM - Maintainability | ⚠️ YES |
| **Total Effort** | | **14-18h** | | |

---

## 🚀 EXECUTION ROADMAP (SEQUENTIAL ORDER)

### **PHASE 1: Error Classification System (3-4 hours)**
**Dependency**: None  
**Must Complete Before**: Worker retry logic

#### Step 1A: Create Job Error Classification
- **File**: `PX-B/app/modules/catalog/job_errors.py`
- **What**: Define error types and retry classification
- **Implementation**:
  ```python
  class JobErrorType(StrEnum):
      SYSTEM_ERROR = "system_error"
      NETWORK_ERROR = "network_error"
      VALIDATION_ERROR = "validation_error"
      CONFLICT_ERROR = "conflict_error"
      CANCELLED_ERROR = "cancelled_error"
      UNKNOWN_ERROR = "unknown_error"
  
  RETRYABLE_ERRORS = {
      JobErrorType.SYSTEM_ERROR,
      JobErrorType.NETWORK_ERROR,
  }
  
  class JobError(Exception):
      def __init__(self, error_type: JobErrorType, message: str, i18n_key: Optional[str] = None):
          self.error_type = error_type
          self.message = message
          self.i18n_key = i18n_key
      
      def is_retryable(self) -> bool:
          return self.error_type in RETRYABLE_ERRORS
  ```

#### Step 1B: Database Migration
```bash
cd PX-B
alembic revision --autogenerate -m "add_error_classification_to_jobs"
```
- **Add columns to `catalog_jobs`**:
  - `error_type` VARCHAR (nullable)
  - `retry_count` INTEGER (default 0)
  - `last_error_at` DATETIME (nullable)
  - `last_error_message` VARCHAR (nullable)

#### Step 1C: Update `CatalogVariantJob` Model
- Add fields from migration to `PX-B/app/modules/catalog/models.py`
- Add indexes: `error_type`, `last_error_at`

#### Step 1D: Create Specific Error Classes
- File: Still `job_errors.py`
- Classes: `StructureStaleError`, `VariantExplosionError`, `DatabaseError`, `JobCancelledError`, `JobTimeoutError`
- Each must inherit `JobError` and set appropriate `error_type`

#### ✅ Validation Gate
```bash
cd PX-B
pytest tests/test_job_error_classification.py -v
```
All tests pass ✓

---

### **PHASE 2: Worker Retry Logic (2-3 hours)**
**Dependency**: Phase 1 (Error Classification)  
**Must Complete Before**: Nothing, but unblocks production reliability

#### Step 2A: Update Worker Execution
- **File**: `PX-B/app/modules/catalog/worker.py` (or create if doesn't exist)
- **Implementation Pattern**:
  ```python
  import time
  from app.modules.catalog.job_errors import JobError, RETRYABLE_ERRORS
  
  MAX_RETRIES = 3
  RETRY_DELAYS = [2, 5, 10]  # seconds with exponential backoff
  
  def _execute_with_retry(job_id: UUID) -> None:
      for attempt in range(MAX_RETRIES):
          try:
              execute_catalog_variant_job(job_id)
              return  # Success!
          except JobError as exc:
              if not exc.is_retryable():
                  logger.error(f"Job {job_id}: Non-retryable {exc.error_type}")
                  raise
              
              if attempt == MAX_RETRIES - 1:
                  logger.error(f"Job {job_id}: Max retries exceeded")
                  raise
              
              delay = RETRY_DELAYS[attempt]
              jitter = random.uniform(0, delay * 0.1)  # 10% jitter
              logger.warning(f"Job {job_id}: Retry {attempt + 1}/{MAX_RETRIES} in {delay + jitter:.1f}s")
              time.sleep(delay + jitter)
          except Exception as exc:
              # Wrap unexpected errors
              wrapped = wrap_error(exc, "Job execution")
              if wrapped.is_retryable() and attempt < MAX_RETRIES - 1:
                  delay = RETRY_DELAYS[attempt]
                  logger.warning(f"Job {job_id}: Unexpected error, retry {attempt + 1}/{MAX_RETRIES}")
                  time.sleep(delay)
              else:
                  raise wrapped
  ```

#### Step 2B: Update Job Lifecycle Tracking
- Modify `_mark_catalog_variant_job()` in `service.py`
- Add parameters: `error_type`, `retry_count`
- Update DB record with error classification

#### Step 2C: Wire into Job Execution
- Update `execute_catalog_variant_job()` to call `_execute_with_retry()`
- Catch and classify ALL exceptions
- Update job status with error type on failure

#### ✅ Validation Gate
```bash
cd PX-B
pytest tests/test_worker_retry_logic.py -v
pytest tests/test_job_error_classification.py -v
```
All tests pass ✓

---

### **PHASE 3: Backend i18n Integration (3-4 hours)**
**Dependency**: Can be done in parallel, but better after Phase 1  
**Must Complete Before**: Production deployment

#### Step 3A: Create i18n Keys Module
- **File**: `PX-B/app/core/i18n_keys.py`
- **Structure**:
  ```python
  class CatalogVariantErrorKeys:
      ACTIVE_JOB_EXISTS = "catalog.variant.errors.active_job_exists"
      STRUCTURE_STALE_VERSION = "catalog.variant.errors.structure_stale_version"
      EXPLOSION_RISK = "catalog.variant.errors.explosion_risk"
      JOB_TIMEOUT = "catalog.variant.errors.job_timeout"
      JOB_CANCELLED = "catalog.variant.errors.job_cancelled"
      GENERATION_FAILED = "catalog.variant.errors.generation_failed"
      # ... all other variant error keys
  
  class CatalogVariantUIKeys:
      STATUS_QUEUED = "catalog.variant.ui.status.queued"
      STATUS_RUNNING = "catalog.variant.ui.status.running"
      STATUS_COMPLETED = "catalog.variant.ui.status.completed"
      # ... all UI keys
  ```
- Copy all keys from `I18N_IMPLEMENTATION.md` section "Phase 1: Backend i18n Keys"

#### Step 3B: Update Error Classes
- **File**: `PX-B/app/exceptions/errors.py`
- Add `i18n_key` parameter to ALL error classes
- Update error response to include `{"i18n_key": "..."}`
- Example:
  ```python
  class CatalogVariantJobActiveError(AppException):
      def __init__(self, product_id: str):
          super().__init__(
              status_code=400,
              code="ACTIVE_JOB_EXISTS",
              detail="A variant job is already running for this product.",
              i18n_key="catalog.variant.errors.active_job_exists",
              error_data={"product_id": product_id}
          )
  ```

#### Step 3C: Update Router Responses
- **File**: `PX-B/app/modules/catalog/router.py`
- All error responses must include `i18n_key`
- Add context data for variable substitution
- Example error response:
  ```json
  {
    "detail": "Too many variants would be created.",
    "error": {
      "code": "EXPLOSION_RISK",
      "i18n_key": "catalog.variant.errors.explosion_risk",
      "context": {"projected": 50000, "limit": 5000}
    },
    "request_id": "req-123"
  }
  ```

#### Step 3D: Frontend Translation Integration
- **File**: `PX-F/px/lib/catalog/i18n.ts` (already exists, verify)
- Verify all keys from backend are translated
- Add new variant error keys to:
  - `public/locales/en/translations.json`
  - `public/locales/es/translations.json`
  - `public/locales/fr/translations.json`
  - `public/locales/de/translations.json`
- Update `lib/catalog/error-handler.ts` to use `i18n_key`

#### ✅ Validation Gate
```bash
cd PX-B
pytest tests/test_i18n_keys.py -v

cd ../PX-F/px
npm run i18n:check
npm test -- --grep "i18n"
```
All tests pass ✓

---

### **PHASE 4: Test Coverage (2-4 hours)**
**Dependency**: Phases 1-3 complete  
**Must Complete Before**: Code review/merge

#### Step 4A: Error Classification Tests
- **File**: `PX-B/tests/test_job_error_classification.py`
- **What to Test**:
  - ✅ Retryable vs non-retryable classification correct
  - ✅ Specific error subtypes instantiate correctly
  - ✅ Error details captured (message, i18n_key, context)
  - ✅ Job record updated with error_type in DB
  - ✅ wrap_error() function works for unexpected errors

#### Step 4B: Worker Retry Logic Tests
- **File**: `PX-B/tests/test_worker_retry_logic.py`
- **What to Test**:
  - ✅ Retryable errors trigger retry (up to MAX_RETRIES)
  - ✅ Non-retryable errors fail immediately
  - ✅ Backoff delays increase exponentially
  - ✅ Jitter added to prevent thundering herd
  - ✅ Job status transitions correctly through retries
  - ✅ Logs capture retry attempts and reasons

#### Step 4C: i18n Integration Tests
- **File**: `PX-B/tests/test_i18n_integration.py`
- **What to Test**:
  - ✅ Error responses include i18n_key
  - ✅ Context data passed for variable substitution
  - ✅ All i18n keys defined in keys module
  - ✅ No hardcoded error strings in responses

- **File**: `PX-F/px/app/components/__tests__/error-handler.test.tsx`
- **What to Test**:
  - ✅ Error i18n_key parsed and translated
  - ✅ Context variables interpolated correctly
  - ✅ Missing translations handled gracefully
  - ✅ Error UI displays localized message

#### Step 4D: Backend Full Test Suite
```bash
cd PX-B
pytest tests/ -v --cov=app
# Must achieve:
# - 80%+ coverage on modules/catalog
# - 90%+ coverage on error classification
# - All variant job lifecycle tests pass
```

#### Step 4E: Frontend Full Test Suite
```bash
cd PX-F/px
npm test -- --coverage
# Must achieve:
# - 70%+ coverage on lib/catalog
# - All VariantStructureStudio tests pass
# - All error handling tests pass
```

#### ✅ Validation Gate
```bash
# BACKEND
cd PX-B
pytest tests/ -v
black --check app/
mypy app/ --ignore-missing-imports
flake8 app/ --max-line-length=100

# FRONTEND
cd ../PX-F/px
npm test
npm run lint
npm run typecheck
npm run i18n:check
```
All pass ✓

---

### **PHASE 5: Dead Code Audit (2-3 hours)**
**Dependency**: Phases 1-4 complete (last to avoid conflicts)

#### Step 5A: Backend Dead Code
```bash
cd PX-B
# Use vulture or manual inspection
vulture app/ --min-confidence 80

# Look for:
- Unused imports
- Unused functions in service.py
- Unused repository functions
- Duplicate error handling code
```

#### Step 5B: Frontend Dead Code
```bash
cd PX-F/px
# Use ESlint
npm run lint -- --fix

# Manual check for:
- Unused React components
- Unused hooks
- Unused utility functions
- Dead CSS classes
```

#### ✅ Validation Gate
```bash
# All unused code removed
# All linters pass
# All tests still pass after cleanup
```

---

## ✅ PRODUCTION READINESS CHECKLIST

**BEFORE MERGING ANY CODE, VERIFY ALL:**

### Code Quality
- [ ] All code follows existing patterns in architecture
- [ ] No hardcoded strings (use i18n keys)
- [ ] No hardcoded values (use config/settings)
- [ ] Proper type hints on all functions
- [ ] Docstrings on public functions
- [ ] Error handling comprehensive (no bare `except:`)

### Testing
- [ ] All unit tests pass: `pytest tests/ -v`
- [ ] All integration tests pass
- [ ] Frontend tests pass: `npm test`
- [ ] Coverage ≥ 80% for modified files
- [ ] Manual testing of error scenarios:
  - Transient failures (retry and succeed)
  - Permanent failures (fail fast)
  - Network timeouts (classified correctly)
  - Concurrency conflicts (handled properly)

### Type Safety
- [ ] TypeScript strict mode passes: `npm run typecheck`
- [ ] Python type hints verified: `mypy app/`
- [ ] No `any` types in critical paths
- [ ] No `# type: ignore` without justification

### Linting & Formatting
- [ ] Python: `black app/`, `flake8 app/`, `pylint app/`
- [ ] TypeScript: `npm run lint`
- [ ] No style warnings in either language
- [ ] Imports properly organized (isort for Python)

### Documentation
- [ ] Updated WHAT-DONE.md with completed work
- [ ] Added comments for non-obvious logic
- [ ] Docstrings on new error types
- [ ] Updated README if new setup steps needed

### i18n Compliance
- [ ] No user-facing strings hardcoded
- [ ] All error messages use i18n keys
- [ ] All UI text uses i18n keys
- [ ] Translations complete for: en, es, fr, de
- [ ] `npm run i18n:check` passes

### Database Changes
- [ ] Alembic migration created (if needed)
- [ ] Migration tested locally: `alembic upgrade head`
- [ ] Backward compatible (can rollback)
- [ ] Indexes added for performance-critical queries

### Performance
- [ ] No N+1 queries in critical paths
- [ ] Retry backoff doesn't exceed 30 seconds total
- [ ] Job execution cancellation is responsive (<5s)
- [ ] SSE heartbeat interval 15-30 seconds

### Security
- [ ] No credentials in logs
- [ ] Rate limiting enforced on public endpoints
- [ ] Store ownership validated on all mutating endpoints
- [ ] User input validated before DB operations
- [ ] Error messages don't leak internal details

### Observability
- [ ] All errors logged with request_id
- [ ] Job state transitions logged
- [ ] Retry attempts logged
- [ ] Metrics endpoint working: `/metrics`

---

## 🧪 CRITICAL TEST PATTERNS

### Backend Error Classification Test Pattern
```python
def test_retryable_error_classification():
    error = DatabaseError("Connection timeout", cause=Exception())
    assert error.is_retryable() is True
    assert error.error_type == JobErrorType.SYSTEM_ERROR
    assert error.i18n_key == "catalog.variant.errors.database_error"

def test_non_retryable_error_classification():
    error = VariantExplosionError("Too many variants", 50000, 5000)
    assert error.is_retryable() is False
    assert error.error_type == JobErrorType.VALIDATION_ERROR
```

### Backend i18n Test Pattern
```python
def test_error_response_includes_i18n_key():
    response = client.post("/catalog/admin/products/{id}/variants/generate", ...)
    assert response.json()["error"]["i18n_key"] == "catalog.variant.errors...."
    assert "context" in response.json()["error"]
```

### Frontend i18n Test Pattern
```tsx
test("translates error message from i18n_key", () => {
  const error = { code: "EXPLOSION_RISK", i18n_key: "catalog.variant.errors.explosion_risk" };
  const translated = translateCatalogKey(error.i18n_key);
  expect(translated).toBe("Too many variants would be created.");
});
```

### Worker Retry Test Pattern
```python
def test_worker_retries_transient_failures():
    job = create_test_job()
    with patch("execute_catalog_variant_job", side_effect=[Exception(), None]):
        _execute_with_retry(job.id)  # Should succeed on retry
    
    updated_job = db.query(CatalogVariantJob).get(job.id)
    assert updated_job.retry_count == 1
    assert updated_job.status == "COMPLETED"
```

---

## 🚨 COMMON MISTAKES TO AVOID

### ❌ DON'T DO THIS

1. **Skip Tests**
   - ❌ "I'll test manually"
   - ✅ Automated tests required, manual testing is bonus

2. **Hardcode Strings**
   - ❌ `raise Exception("Product not found")`
   - ✅ Use i18n keys: `i18n_key="catalog.product.errors.not_found"`

3. **Catch All Exceptions**
   - ❌ `except Exception: pass`
   - ✅ Catch specific errors, classify them

4. **Retry Everything**
   - ❌ Retry validation errors (permanent failures)
   - ✅ Only retry transient errors (system, network)

5. **Skip Type Hints**
   - ❌ `def process(job):`
   - ✅ `def process(job: CatalogVariantJob) -> None:`

6. **Ignore Backward Compatibility**
   - ❌ Change error response format
   - ✅ Add new fields, keep old ones

7. **Mix Concerns**
   - ❌ Error handling in router
   - ✅ Error handling in service, router uses it

8. **Forget Logging**
   - ❌ Silent failures
   - ✅ Log with request_id, error type, context

9. **Incomplete Translations**
   - ❌ Only English
   - ✅ All supported locales (en, es, fr, de, ar, hi, etc.)

10. **Database Changes Without Migration**
    - ❌ Modify model without alembic
    - ✅ Create migration first, update model

---

## 📋 FILE MODIFICATION CHECKLIST

### Backend Files to Create/Modify

| File | Status | Action |
|------|--------|--------|
| `app/modules/catalog/job_errors.py` | NEW | Create with all error types |
| `app/core/i18n_keys.py` | NEW | Create with all i18n keys |
| `app/modules/catalog/models.py` | MODIFY | Add error_type, retry_count fields to CatalogVariantJob |
| `app/modules/catalog/worker.py` | MODIFY | Add retry logic with exponential backoff |
| `app/modules/catalog/service.py` | MODIFY | Update `_mark_catalog_variant_job`, catch/classify errors |
| `app/exceptions/errors.py` | MODIFY | Add i18n_key to all error classes |
| `app/modules/catalog/router.py` | MODIFY | Include i18n_key in error responses |
| `alembic/versions/[TIMESTAMP].py` | NEW | Migration for job_errors fields |
| `tests/test_job_error_classification.py` | NEW | Error classification tests |
| `tests/test_worker_retry_logic.py` | NEW | Retry logic tests |
| `tests/test_i18n_integration.py` | NEW | i18n integration tests |

### Frontend Files to Create/Modify

| File | Status | Action |
|------|--------|--------|
| `lib/catalog/i18n.ts` | VERIFY | All new variant error keys present |
| `public/locales/en/translations.json` | MODIFY | Add new error translations |
| `public/locales/es/translations.json` | MODIFY | Add new error translations |
| `public/locales/fr/translations.json` | MODIFY | Add new error translations |
| `public/locales/de/translations.json` | MODIFY | Add new error translations |
| `lib/catalog/error-handler.ts` | VERIFY | Uses i18n_key from backend |
| `app/components/__tests__/error-handler.test.tsx` | NEW | i18n error handling tests |

---

## 🔄 DEPENDENCY GRAPH

```
Phase 1: Error Classification
    ↓
Phase 2: Worker Retry Logic ← (Depends on Phase 1)
    ↓
Phase 3: Backend i18n (Can be parallel, but cleaner after Phase 1)
    ↓
Phase 4: Test Coverage ← (Depends on Phases 1-3)
    ↓
Phase 5: Dead Code Audit ← (Depends on Phases 1-4, last to avoid conflicts)
    ↓
PRODUCTION READY ✓
```

---

## 🎓 REFERENCE DOCUMENTS

When implementing, READ THESE IN ORDER:

1. **CODE_AGENT_QUICKSTART.md** (5 min) - High-level overview
2. **IMPLEMENTATION_GUIDE.md** (10 min) - Priority matrix and phases
3. **CODE_AGENT_COMPREHENSIVE_GUIDE.md** (THIS FILE) - Detailed execution plan
4. **I18N_IMPLEMENTATION.md** - i18n specific details
5. **ERROR_CLASSIFICATION.md** - Error handling specifics
6. **variant-domain-contract.md** - Business rules and guarantees
7. **WHAT-DONE.md** - Track progress

---

## 📞 VALIDATION GATES (MUST PASS)

### Gate 1: After Phase 1 (Error Classification)
```bash
cd PX-B && pytest tests/test_job_error_classification.py -v
# Expected: All tests pass
# Verify: JobErrorType enum exists, is_retryable() works, DB migration applied
```

### Gate 2: After Phase 2 (Worker Retry)
```bash
cd PX-B && pytest tests/test_worker_retry_logic.py -v
pytest tests/test_catalog_variant_*.py -v  # Ensure existing tests still pass
# Expected: All tests pass, including existing variant tests
```

### Gate 3: After Phase 3 (i18n Backend)
```bash
cd PX-B && pytest tests/test_i18n_integration.py -v
cd ../PX-F/px && npm run i18n:check && npm test
# Expected: No hardcoded strings, all translations present
```

### Gate 4: After Phase 4 (Test Coverage)
```bash
cd PX-B
pytest tests/ -v --cov=app --cov-fail-under=80
black --check app/ && mypy app/ && flake8 app/

cd ../PX-F/px
npm test -- --coverage
npm run lint && npm run typecheck
# Expected: 80%+ coverage, 0 lint errors, all types pass
```

### Gate 5: After Phase 5 (Dead Code Audit)
```bash
cd PX-B && pytest tests/ -v  # Everything still works
cd ../PX-F/px && npm test && npm run lint
# Expected: Cleaner codebase, all tests pass
```

### Final Gate: Production Readiness
```bash
# Backend
cd PX-B
pytest tests/ -v
black --check app/
mypy app/ --ignore-missing-imports
flake8 app/ --max-line-length=100
# Build should succeed
make build

# Frontend
cd ../PX-F/px
npm test
npm run lint
npm run typecheck
npm run i18n:check
npm run build
# Build should succeed, no warnings

# Result: PRODUCTION READY ✅
```

---

## 🎯 SUCCESS CRITERIA

**Implementation is complete when:**

- ✅ All 3 blocking issues fixed (error classification, retry, i18n)
- ✅ Error classification tests ≥ 95% pass rate
- ✅ Worker retry logic tested with transient + permanent failures
- ✅ All user-facing text uses i18n keys
- ✅ All 4 languages have complete translations
- ✅ No hardcoded error strings in backend
- ✅ No hardcoded UI strings in frontend
- ✅ 80%+ test coverage on modified code
- ✅ Zero linting errors in both languages
- ✅ All type checks pass (mypy + TypeScript)
- ✅ WHAT-DONE.md updated with completed phases
- ✅ All validation gates pass
- ✅ Code reviewed for production quality

**Once all criteria met: System is production-ready.**

---

## 📝 FINAL NOTES FOR CODE AGENTS

- **Do NOT skip tests** — They catch real bugs
- **Do NOT hardcode strings** — They must be translatable
- **Do NOT ignore errors** — Classify and handle properly
- **Do NOT assume transient failures** — Distinguish permanent vs transient
- **Do NOT deploy without validation gates** — They verify correctness
- **Do read WHAT-DONE.md** — Update it as you progress
- **Do ask questions** — Better now than in production

---

**Start with Phase 1. Complete validation gate. Then move to Phase 2.**

**This is production code. Treat it as such.**
