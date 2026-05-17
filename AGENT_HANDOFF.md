# 🤝 AGENT HANDOFF & CONTINUITY DOCUMENT
## Current Implementation Status & Architecture Context

**Last Updated**: 2026-05-17  
**Next Agent Should Read**: This file first, then 00_START_HERE.md  
**Purpose**: Ensure continuity between agents and preserve architectural reasoning

---

## 🎯 CURRENT PROJECT STATE (as of 2026-05-17)

### Overall Status: 65-70% Complete
- ✅ Core architecture implemented
- ✅ Frontend i18n infrastructure complete
- ✅ Database models finalized
- ❌ 3 critical blockers remain (error classification, retry logic, backend i18n)

### What's Done
1. **Core Variant Domain** (100%)
   - Canonical identity helpers
   - Structure lifecycle (idle, dirty, synced)
   - Variant lifecycle (active, archived, deleted)
   - Job status guards

2. **SSE Infrastructure** (80%)
   - Event bus with monotonic sequences
   - Authenticated SSE endpoints
   - Event history (in-memory + DB-backed)
   - Replay capability

3. **Explosion Protection** (100%)
   - 10 options max, 100 values per option
   - Tier-aware quotas (Basic 5K, Pro 50K)
   - Readiness checks before job submission

4. **Performance Foundations** (100%)
   - Job cancellation API
   - Job timeout cleanup
   - Paginated variant listing
   - Subscription-based scaling support

### What's NOT Done (3 Critical Blockers)
1. **Error Classification** (0%)
   - ❌ JobErrorType enum not created
   - ❌ Error_type field not added to CatalogVariantJob
   - ❌ No retry classification logic

2. **Worker Retry Logic** (0%)
   - ❌ No _execute_with_retry() function
   - ❌ No exponential backoff implementation
   - ❌ Transient failures cause permanent job failures

3. **Backend i18n Integration** (0%)
   - ❌ app/core/i18n_keys.py not created
   - ❌ Error classes don't return i18n_key
   - ❌ Backend errors return English strings only

---

## 📊 IMPLEMENTATION PHASES TIMELINE

### Phase 1: Error Classification (NEXT - 3-4 hours)
**Status**: Not started  
**Blocker**: YES - Must complete before Phase 2  
**Files**:
- Create: `PX-B/app/modules/catalog/job_errors.py`
- Modify: `PX-B/app/modules/catalog/models.py`
- Create: Alembic migration

**Validation Gate**: `pytest tests/test_job_error_classification.py -v`

### Phase 2: Worker Retry Logic (2-3 hours)
**Status**: Not started  
**Blocker**: YES - Depends on Phase 1  
**Files**:
- Modify: `PX-B/app/modules/catalog/worker.py`
- Modify: `PX-B/app/modules/catalog/service.py`

**Validation Gate**: `pytest tests/test_worker_retry_logic.py -v`

### Phase 3: Backend i18n Integration (3-4 hours)
**Status**: Not started  
**Blocker**: YES - Can be parallel  
**Files**:
- Create: `PX-B/app/core/i18n_keys.py`
- Modify: `PX-B/app/exceptions/errors.py`
- Modify: `PX-B/app/modules/catalog/router.py`

**Validation Gate**: `pytest tests/test_i18n_integration.py -v && npm run i18n:check`

### Phase 4: Test Coverage (2-4 hours)
**Status**: Not started  
**Blocker**: NO (but gates other phases)  
**Files**:
- Create: `PX-B/tests/test_job_error_classification.py`
- Create: `PX-B/tests/test_worker_retry_logic.py`
- Create: `PX-B/tests/test_i18n_integration.py`

**Validation Gate**: `pytest tests/ -v --cov=app --cov-fail-under=80`

### Phase 5: Dead Code Audit (2-3 hours)
**Status**: Not started  
**Blocker**: NO (last phase, for cleanup)  
**Files**: Various (identified via vulture/eslint)

**Validation Gate**: `pytest tests/ -v && npm test`

---

## 🏗️ ARCHITECTURE DECISIONS & REASONING

### Decision 1: Snapshot-Based Job Execution
**Why**: Prevents race conditions when product structure changes during job execution  
**How**: Job captures immutable snapshot of product structure at start, executes against snapshot  
**Important**: Changes to structure during execution don't affect running job  
**File**: `PX-B/app/modules/catalog/service.py:execute_catalog_variant_job()`

### Decision 2: Eventual Consistency for Storefront
**Why**: Admin changes shouldn't immediately affect live storefront  
**How**: Admin saves → validated → generated → published snapshot → storefront syncs  
**Important**: Prevents partial visibility and inventory mismatches  
**File**: `PX-B/app/modules/catalog/service.py:publish_catalog_product_snapshot()`

### Decision 3: SSE for Job Progress (not polling)
**Why**: Real-time updates without client polling  
**How**: Server-Sent Events stream job lifecycle events  
**Important**: Clients stream events, apply locally, reconcile on reconnect  
**File**: `PX-B/app/modules/catalog/variant_events.py`

### Decision 4: Error Classification for Intelligent Retry
**Why**: Can't retry validation errors (permanent), must retry transient errors  
**How**: Classify errors → determine retryability → retry with exponential backoff  
**Important**: This is CRITICAL for production reliability  
**File**: To be created: `PX-B/app/modules/catalog/job_errors.py`

### Decision 5: i18n Keys in Error Responses
**Why**: Frontend needs keys to translate error messages to user's language  
**How**: Backend error responses include `{"i18n_key": "catalog.variant.errors...."}`  
**Important**: Users see errors in their language, not English  
**File**: To be created: `PX-B/app/core/i18n_keys.py`

### Decision 6: Tier-Based Variant Quotas
**Why**: Protect system from variant explosion  
**How**: Basic plan 5K max, Pro plan 50K max variants  
**Important**: Enforced at API layer before job submission  
**File**: `PX-B/app/modules/catalog/service.py:check_variant_explosion_risk()`

---

## 📋 IMPORTANT FILES & MODULES

### Backend Core
- **app/main.py** - FastAPI app setup, middleware pipeline, router registration
- **app/core/config.py** - Settings and environment variables
- **app/core/logging.py** - Structured logging configuration
- **app/exceptions/errors.py** - Exception classes (update to add i18n_key)

### Variant Feature
- **app/modules/catalog/models.py** - SQLModel definitions (CatalogVariantJob, ProductVariant, etc.)
- **app/modules/catalog/repository.py** - Database access layer
- **app/modules/catalog/service.py** - Business logic (MOST COMPLEX)
- **app/modules/catalog/router.py** - API endpoints
- **app/modules/catalog/variant_domain.py** - Shared domain logic (canonical keys, lifecycle)
- **app/modules/catalog/variant_events.py** - SSE event bus

### Job Execution (Will be Modified)
- **app/modules/catalog/worker.py** - Job execution worker (add retry logic here)
- **app/modules/catalog/job_errors.py** - ERROR CLASSIFICATION (CREATE THIS)

### Frontend
- **lib/i18n/** - i18n configuration (COMPLETE, no changes needed)
- **lib/catalog/variant-domain.ts** - Shared domain logic (matches backend)
- **lib/catalog/error-handler.ts** - Error parsing and translation
- **app/components/product-editor/variant-structure-studio.tsx** - Main UI component

---

## 🚨 CRITICAL GOTCHAS & LESSONS LEARNED

### Gotcha 1: N+1 Queries in Variant Loading
**Issue**: Loading 50 variants with 5 options each causes 250 DB queries  
**Solution**: Use `join()` in SQLModel to eagerly load relationships  
**File**: `PX-B/app/modules/catalog/service.py:list_variants_for_product()`  
**Learn More**: Check existing tests for join patterns

### Gotcha 2: Race Condition in Job Status Updates
**Issue**: Two concurrent requests might update job status simultaneously  
**Solution**: Use database transactions with proper locking  
**File**: `PX-B/app/modules/catalog/service.py:_mark_catalog_variant_job()`  
**Pattern**: Always use `session.flush()` after critical updates

### Gotcha 3: i18n Keys Must Match Frontend
**Issue**: Backend error key "catalog.variant.errors.explosion_risk" must exist in frontend translations  
**Solution**: Maintain 1:1 mapping between backend keys and frontend translation files  
**File**: `lib/catalog/i18n.ts`  
**Pattern**: Check BOTH sides when adding new errors

### Gotcha 4: Snapshot Must Be Immutable
**Issue**: If job modifies snapshot during execution, causes data corruption  
**Solution**: Snapshot is READ-ONLY during job execution  
**File**: `PX-B/app/modules/catalog/service.py:execute_catalog_variant_job()`  
**Important**: Never modify `input_snapshot_json` after job starts

### Gotcha 5: Job Cancellation Must Be Cooperative
**Issue**: Can't force-kill worker process mid-variant-creation  
**Solution**: Worker checks `job.cancelled` flag at each loop iteration  
**File**: `PX-B/app/modules/catalog/service.py:generate_missing_variants_for_product()`  
**Pattern**: Check cancellation in loops, clean up resources

### Gotcha 6: Exponential Backoff Needs Jitter
**Issue**: All failed jobs retry at same time → thundering herd  
**Solution**: Add randomized jitter (±10%) to retry delays  
**File**: To be created: `PX-B/app/modules/catalog/worker.py`  
**Pattern**: `delay + random.uniform(0, delay * 0.1)`

---

## 🔄 PROJECT FLOW & CONVENTIONS

### Code Organization Pattern
```
app/
├── core/              # Shared infrastructure (config, logging, i18n keys)
├── middleware/        # HTTP middleware (auth, CORS, etc.)
├── exceptions/        # Exception definitions
├── modules/           # Feature modules (auth, catalog, stores, users)
│   └── catalog/
│       ├── models.py  # SQLModel definitions
│       ├── schemas.py # Request/response schemas
│       ├── repository.py  # DB access
│       ├── service.py  # Business logic (largest file)
│       ├── router.py   # API endpoints
│       ├── worker.py   # Background job execution
│       └── ...
└── db/                # Database session, migrations
```

### Error Handling Pattern
```python
# In service.py (business logic)
try:
    # Do work
except SpecificError as e:
    # Classify error
    raise JobError(JobErrorType.VALIDATION_ERROR, message, i18n_key)

# In router.py (API endpoints)
try:
    result = service.do_something()
except AppException as e:
    # Exception handler converts to JSON response with i18n_key
```

### Testing Pattern
```python
# tests/test_catalog_variant_*.py
def test_happy_path():
    # Arrange
    job = create_test_job()
    # Act
    result = execute_variant_job(job.id)
    # Assert
    assert result.status == "COMPLETED"

def test_error_path():
    # Should test that specific error is raised/classified
    with pytest.raises(JobError) as exc_info:
        execute_variant_job(bad_job.id)
    assert exc_info.value.error_type == JobErrorType.VALIDATION_ERROR
```

### i18n Pattern
```python
# Backend (error response)
raise CatalogVariantJobActiveError(
    product_id=str(product_id),
    # Error class automatically includes:
    # - i18n_key="catalog.variant.errors.active_job_exists"
    # - message="A variant job is already running"
)

# Frontend (error display)
const translated = t(error.i18n_key, { context: error.context });
```

---

## 📌 CURRENT AGENT THOUGHT PROCESS & DIRECTION

### Why These 3 Phases Are Critical
1. **Error Classification** - Without it, transient failures (DB hiccup, network timeout) cause permanent job failures instead of retrying. Production WILL fail.
2. **Retry Logic** - Without it, temporary issues become permanent, reducing success rate. Users see "job failed" for random reasons.
3. **i18n Backend** - Without it, non-English users see English error messages. Multi-language support is incomplete.

### Implementation Strategy
- Do Phase 1 (error classification) first - it's the foundation
- Phase 2 (retry logic) depends on Phase 1 being complete
- Phase 3 (i18n) can be done in parallel but cleaner after Phase 1
- Phase 4 (tests) validates that 1-3 work correctly
- Phase 5 (cleanup) is last after everything is stable

### Testing Strategy
- Each phase has validation gates that MUST pass
- Tests are not optional - they catch real bugs
- Test patterns provided in CODE_AGENT_COMPREHENSIVE_GUIDE.md
- Coverage ≥80% required for production

---

## ⏳ PENDING TASKS & BLOCKERS

### Immediate Next Tasks (in order)
1. ✅ Create comprehensive guides (DONE - this sprint)
2. ⏳ **NEXT: Implement Phase 1 (Error Classification) - 3-4 hours**
   - Files: job_errors.py, alembic migration, model update
   - Validation: test_job_error_classification.py passes
3. ⏳ Implement Phase 2 (Worker Retry Logic) - 2-3 hours
4. ⏳ Implement Phase 3 (Backend i18n) - 3-4 hours
5. ⏳ Implement Phase 4 (Test Coverage) - 2-4 hours
6. ⏳ Implement Phase 5 (Dead Code Audit) - 2-3 hours

### Known Blockers
- **Blocker 1**: Error classification must be done before retry logic (dependency)
- **Blocker 2**: Database migration must be applied before code changes (order matters)
- **Blocker 3**: All 4 languages must have i18n translations (en, es, fr, de)

### Known Limitations
- Worker is in-process for now (scales to ~100 jobs/sec)
- In-memory event bus retains last 1000 events (older events from DB)
- Job timeout detection is every 5 minutes (not real-time)

---

## 🔍 HOW TO UNDERSTAND THIS CODEBASE QUICKLY

### For a New Agent: Read in This Order
1. This file (AGENT_HANDOFF.md) - 10 min - Get context
2. 00_START_HERE.md - 5 min - Understand guides
3. CODE_AGENT_COMPREHENSIVE_GUIDE.md (your phase) - 30 min - Get implementation steps
4. variant-domain-contract.md - 15 min - Understand business rules
5. Existing code in app/modules/catalog/ - 30-60 min - See patterns

### For a Returning Agent: Just Read
1. This file (AGENT_HANDOFF.md) - What's been done, what's pending
2. WHAT-DONE.md - What the last agent accomplished
3. Jump to CODE_AGENT_COMPREHENSIVE_GUIDE.md for next phase

### For a Curious Agent: Check
- variant-domain-contract.md - "Why are variants designed this way?"
- technical-architecture-spec.md - "How does the architecture work?"
- VARIANTS_FEATURE_STRUCTURE.md - "Where is everything in the code?"

---

## 🔐 IMPORTANT REMINDERS FOR NEXT AGENT

### DO ✅
- ✅ Write tests first (test patterns provided)
- ✅ Follow validation gates (they catch bugs)
- ✅ Check DO_NOT_FORGET.md before every commit (115+ items)
- ✅ Update WHAT-DONE.md after each phase (track progress)
- ✅ Use CODE_AGENT_COMPREHENSIVE_GUIDE.md step-by-step (don't improvise)
- ✅ Check all 4 language translations (en, es, fr, de)
- ✅ Run full test suite after changes (don't skip)

### DON'T ❌
- ❌ Skip database migrations (data corruption risk)
- ❌ Hardcode error strings (breaks i18n)
- ❌ Retry validation errors (permanent failures)
- ❌ Miss type hints (silent runtime errors)
- ❌ Skip tests ("I'll test manually" fails)
- ❌ Change error response format (breaks frontend)
- ❌ Deploy without validation gates (production failures)

---

## 📞 IF YOU GET STUCK

### Question: "What file do I modify?"
**Answer**: Check CODE_AGENT_COMPREHENSIVE_GUIDE.md, your phase section

### Question: "What should I test?"
**Answer**: Check TEST PATTERNS section in COMPREHENSIVE_GUIDE

### Question: "Did I miss something?"
**Answer**: Check DO_NOT_FORGET.md - 115+ item checklist

### Question: "Why is the system designed this way?"
**Answer**: Check variant-domain-contract.md - business rules

### Question: "Where is this code located?"
**Answer**: Check VARIANTS_FEATURE_STRUCTURE.md - file organization

### Question: "What validation gates must pass?"
**Answer**: Check CODE_AGENT_COMPREHENSIVE_GUIDE.md - validation section

---

## 🎯 SUCCESS DEFINITION FOR NEXT AGENT

**You're done when:**
- ✅ All 5 phases implemented
- ✅ All validation gates pass
- ✅ 80%+ test coverage
- ✅ All 4 languages have complete i18n
- ✅ Zero hardcoded error strings
- ✅ Linting + type checking pass
- ✅ WHAT-DONE.md fully updated
- ✅ Code reviewed for production quality

---

## 📋 FILES TO READ NEXT (BY ROLE)

### If you're implementing Phase 1:
- CODE_AGENT_COMPREHENSIVE_GUIDE.md § PHASE 1
- ERROR_CLASSIFICATION.md (detailed spec)
- IMPLEMENTATION_CHECKLIST.md § BLOCKER 2

### If you're implementing Phase 2:
- CODE_AGENT_COMPREHENSIVE_GUIDE.md § PHASE 2
- ERROR_CLASSIFICATION.md § Worker Retry Logic section
- IMPLEMENTATION_CHECKLIST.md § BLOCKER 3

### If you're implementing Phase 3:
- CODE_AGENT_COMPREHENSIVE_GUIDE.md § PHASE 3
- I18N_IMPLEMENTATION.md (detailed spec)
- IMPLEMENTATION_CHECKLIST.md § BLOCKER 1

### If you're implementing Phases 4-5:
- CODE_AGENT_COMPREHENSIVE_GUIDE.md § PHASE 4 or 5
- QUICK_REFERENCE.md (for quick lookup)
- DO_NOT_FORGET.md (safety checklist)

---

## ✨ FINAL NOTES

This project is well-architected and documented. The 3 blocking issues are isolated, testable, and well-defined. Each phase has:
- Clear acceptance criteria
- Validation gates
- Test patterns
- Code examples

**The system is designed to be implementable by any competent agent following the guides.**

Good luck! 🚀

---

**→ For next implementation steps, read: CODE_AGENT_COMPREHENSIVE_GUIDE.md**
