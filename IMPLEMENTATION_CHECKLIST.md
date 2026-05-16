# Implementation Checklist for Code Agents

**Purpose**: Track progress through all implementation phases  
**Use**: Check off each item as you complete it  
**Update**: Maintain this as you work - it's your progress tracker

---

## 📚 Documentation Files (Read These First)

- [ ] Read `CODE_AGENT_QUICKSTART.md` (5 min orientation)
- [ ] Read `IMPLEMENTATION_GUIDE.md` (priority matrix overview)
- [ ] Read detailed guide for your current task

---

## 🔴 BLOCKER 1: i18n Framework Implementation

### Backend: Create i18n Keys
- [ ] Create `PX-B/app/core/i18n_keys.py`
  - [ ] `CatalogVariantErrorKeys` class with all error keys
  - [ ] `CatalogVariantUIKeys` class with all UI strings
  - [ ] Import in `__init__.py` for easy access
- [ ] Verify all error messages have corresponding keys (check `I18N_IMPLEMENTATION.md` list)

### Backend: Update Error Classes
- [ ] Update `PX-B/app/exceptions/errors.py`
  - [ ] Add `message_key` parameter to `CatalogVariantJobActiveError`
  - [ ] Add `message_key` to `ProductStructureStaleVersionError`
  - [ ] Add `message_key` to `ProductStructureStaleHashError`
  - [ ] Add `message_key` to all other catalog variant errors
  - [ ] All errors return `{"i18n_key": "...", "context": {...}}`

### Backend: Update Router Responses
- [ ] Update `PX-B/app/modules/catalog/router.py`
  - [ ] All error responses include `i18n_key`
  - [ ] Context data for variable substitution included
  - [ ] Test error responses with curl/Postman

### Frontend: Install Dependencies
- [ ] Run `cd PX-F/px && npm install i18next react-i18next`
- [ ] Update `PX-F/px/package.json` to include both packages
- [ ] Run `npm install` to verify no conflicts

### Frontend: Create i18n Config
- [ ] Create `PX-F/px/lib/i18n/index.ts`
  - [ ] Initialize i18next with resources
  - [ ] Set fallback language to English
  - [ ] Configure detection (localStorage → navigator)
- [ ] Create `PX-F/px/lib/i18n/config.ts` if separate from index

### Frontend: Create Translation Files
- [ ] Create directory structure:
  ```
  public/locales/
  ├── en/
  │   └── translations.json
  ├── es/
  │   └── translations.json
  ├── fr/
  │   └── translations.json
  └── de/
      └── translations.json
  ```
- [ ] Create `en/translations.json` with all keys from `I18N_IMPLEMENTATION.md`
- [ ] Create `es/translations.json` (Spanish)
- [ ] Create `fr/translations.json` (French)
- [ ] Create `de/translations.json` (German)
- [ ] Verify all keys present in all files

### Frontend: Integrate i18n in App
- [ ] Initialize i18n in `PX-F/px/app/layout.tsx` or root component
- [ ] Verify i18n is loaded before components render
- [ ] Test that `i18n.language` changes correctly

### Frontend: Update Components
- [ ] Update `PX-F/px/app/components/VariantStructureStudio.tsx`
  - [ ] Use `useTranslation()` hook
  - [ ] Replace hardcoded strings with `t()` calls
  - [ ] Test all buttons show translated text
- [ ] Update `PX-F/px/lib/catalog/error-handler.ts`
  - [ ] Parse `i18n_key` from error responses
  - [ ] Use `t()` to translate keys
  - [ ] Include context in translations
- [ ] Update other variant components

### Testing: i18n
- [ ] Backend: `cd PX-B && pytest tests/ -v -k "i18n or error"`
  - [ ] All error responses have `i18n_key`
  - [ ] Context data included where needed
- [ ] Frontend: `cd PX-F/px && npm test -- --grep "i18n"`
  - [ ] i18n config initializes
  - [ ] Translations load correctly
  - [ ] Missing keys handled gracefully
- [ ] Frontend: `npm run i18n:check` passes
- [ ] Manual test: Change browser language, verify UI updates

### Validation: i18n
- [ ] No hardcoded error strings in backend
- [ ] No hardcoded UI strings in frontend
- [ ] All i18n keys defined and used
- [ ] Translation files complete for all languages

---

## 🔴 BLOCKER 2: Error Classification System

### Backend: Create Job Error Classes
- [ ] Create `PX-B/app/modules/catalog/job_errors.py`
  - [ ] `JobErrorType` enum with all error types
  - [ ] `RETRYABLE_ERRORS` set definition
  - [ ] `NON_RETRYABLE_ERRORS` set definition
  - [ ] `JobError` base class with all fields
  - [ ] Specific error classes:
    - [ ] `StructureStaleError`
    - [ ] `VariantExplosionError`
    - [ ] `DatabaseError`
    - [ ] `JobCancelledError`
    - [ ] `JobTimeoutError`
    - [ ] `ResourceExhaustedError`
    - [ ] `InvalidProductError`
  - [ ] `wrap_error()` helper function

### Backend: Update Database Model
- [ ] Create Alembic migration:
  ```bash
  cd PX-B && alembic revision --autogenerate -m "add error classification fields"
  ```
- [ ] Verify migration adds these fields to `catalog_jobs` table:
  - [ ] `error_type` (varchar, nullable)
  - [ ] `retry_count` (int, default 0)
  - [ ] `last_error_at` (datetime, nullable)
  - [ ] `last_error_message` (varchar, nullable)
- [ ] Update `PX-B/app/modules/catalog/models.py`
  - [ ] Add fields to `CatalogVariantJob` class
  - [ ] Add indexes on `error_type` and `last_error_at`
- [ ] Run migration: `alembic upgrade head`

### Backend: Update Service Layer
- [ ] Update `_mark_catalog_variant_job()` function
  - [ ] Accept `error_type` parameter
  - [ ] Accept `retry_count` parameter
  - [ ] Update fields on job object
  - [ ] Update timestamps correctly
- [ ] Update `execute_catalog_variant_job()` function
  - [ ] Import all `JobError` types
  - [ ] Catch `JobError` separately
  - [ ] Catch generic `Exception` as fallback
  - [ ] Call `_mark_catalog_variant_job()` with error_type
  - [ ] Re-raise errors for worker to handle
- [ ] Update `_run_variant_job_phases()`
  - [ ] Catch specific errors and wrap as `JobError`
  - [ ] Provide meaningful error messages
  - [ ] Include context in error details

### Testing: Error Classification
- [ ] Create `PX-B/tests/test_job_error_classification.py`
  - [ ] Test retryable vs non-retryable classification
  - [ ] Test specific error subtypes
  - [ ] Test error details captured correctly
- [ ] Run tests: `pytest tests/test_job_error_classification.py -v`
- [ ] Verify error_type field populated in DB

### Validation: Error Classification
- [ ] All errors are `JobError` subclasses
- [ ] Error types correctly classified
- [ ] Retryable/non-retryable sets accurate
- [ ] All errors have i18n keys from Step 1

---

## 🔴 BLOCKER 3: Worker Retry Logic

### Backend: Update Worker Process
- [ ] Update `PX-B/app/modules/catalog/worker.py`
  - [ ] Import error classification module
  - [ ] Add retry constants:
    - [ ] `MAX_RETRIES = 3`
    - [ ] `RETRY_DELAYS = [2, 5, 10]`
    - [ ] `JOB_POLLING_INTERVAL = 2`
  - [ ] Implement `_execute_with_retry()` method:
    - [ ] Loop up to MAX_RETRIES times
    - [ ] Try to execute job
    - [ ] On JobError: check if retryable
    - [ ] If retryable: sleep and retry
    - [ ] If non-retryable: mark failed and return
    - [ ] Use exponential backoff delays
  - [ ] Update `run()` main loop:
    - [ ] Use `_execute_with_retry()` for all jobs
    - [ ] Don't catch JobError (let fail for retry)
  - [ ] Implement graceful shutdown:
    - [ ] Handle SIGTERM/SIGINT
    - [ ] Log shutdown message
    - [ ] Exit loop cleanly
  - [ ] Add comprehensive logging:
    - [ ] Log each retry attempt
    - [ ] Include error_type in logs
    - [ ] Log backoff delays
    - [ ] Log final success/failure

### Backend: Makefile Updates
- [ ] Ensure `make run-worker` target exists
- [ ] Ensure `make dev-worker` target exists
- [ ] Test targets:
  ```bash
  make run-worker  # Should start worker
  make dev-worker  # Should run with auto-reload
  ```

### Testing: Worker Retry Logic
- [ ] Create `PX-B/tests/test_worker_retry.py`
  - [ ] Mock transient error, verify retries
  - [ ] Mock permanent error, verify fails immediately
  - [ ] Verify exponential backoff timing
  - [ ] Verify cancelled jobs don't retry
  - [ ] Verify max retries enforced
- [ ] Run tests: `pytest tests/test_worker_retry.py -v`

### Manual Testing: Worker
- [ ] Start worker: `make run-worker` in one terminal
- [ ] Create test variant job in another terminal
- [ ] Verify worker picks it up
- [ ] Verify job completes or fails appropriately
- [ ] Check job status in DB

### Validation: Worker Retry Logic
- [ ] Worker doesn't retry non-retryable errors
- [ ] Worker retries transient errors up to MAX_RETRIES
- [ ] Exponential backoff applied correctly
- [ ] Graceful shutdown works
- [ ] All retry attempts logged

---

## 🟡 HIGH PRIORITY: Performance & Observability

### Performance: Optimize N+1 Queries
- [ ] Identify N+1 patterns in `service.py`
  - [ ] Options with values loading
  - [ ] Variants with selections loading
- [ ] Update queries to use joinedload/selectinload
- [ ] Run query profiling: `pytest tests/ --profile=cprofile`
- [ ] Verify query count reduced

### Performance: Rate Limiting
- [ ] Create `PX-B/app/middleware/rate_limit.py`
  - [ ] `RateLimiter` class
  - [ ] Per-user and per-store limiters
- [ ] Add to job endpoints in `router.py`
  - [ ] Check rate limit before creating job
  - [ ] Return 429 if exceeded
- [ ] Test rate limiting

### Observability: Metrics Collection
- [ ] Create `PX-B/app/core/metrics.py`
  - [ ] `JobMetrics` dataclass
  - [ ] `MetricsCollector` class
  - [ ] Tracking methods (start, complete)
  - [ ] Stats aggregation
- [ ] Add metrics endpoint in `router.py`
  - [ ] GET `/admin/metrics/variant-jobs`
  - [ ] Returns stats JSON
- [ ] Decorate job execution with `@track_job_execution`

### Testing & Validation
- [ ] All tests still pass with optimizations
- [ ] N+1 queries eliminated
- [ ] Rate limiting tests passing
- [ ] Metrics collection working

---

## 🧪 Comprehensive Testing

### Backend Tests
- [ ] `cd PX-B && python -m pytest tests/ -v`
  - [ ] All tests pass
  - [ ] No skipped tests
  - [ ] Coverage >80%
- [ ] Type checking: `mypy app/`
- [ ] Linting: `ruff check .`
- [ ] All commands should report 0 errors

### Frontend Tests
- [ ] `cd PX-F/px && npm test`
  - [ ] All tests pass
  - [ ] No warnings
- [ ] Type checking: `npm run typecheck`
- [ ] Linting: `npm run lint`
- [ ] i18n check: `npm run i18n:check`
- [ ] All commands should pass

### Integration Testing
- [ ] Backend + Frontend together:
  - [ ] Create variant job
  - [ ] Verify error messages translated
  - [ ] Verify retry logic on transient error
  - [ ] Verify permanent errors fail immediately

### End-to-End Testing
- [ ] Start backend: `make dev`
- [ ] Start frontend: `cd PX-F/px && npm run dev`
- [ ] Manual user flows:
  - [ ] Generate variants
  - [ ] Encounter error
  - [ ] Verify error shown in correct language
  - [ ] Retry works if transient
  - [ ] Fails appropriately if permanent

---

## 📝 Documentation & Cleanup

### Code Cleanup
- [ ] Remove unused imports
- [ ] Remove debug statements
- [ ] Remove commented-out code
- [ ] Check for dead code (use `vulture` or similar)

### Documentation
- [ ] Update README with i18n setup steps
- [ ] Update architecture docs if needed
- [ ] Add docstrings to new classes/functions
- [ ] Document retry behavior

### Final Validation
- [ ] All tests pass
- [ ] All linting passes
- [ ] All type checks pass
- [ ] No security warnings
- [ ] Code reviewed for consistency

---

## 📊 Update WHAT-DONE.md

After completing all blockers, update `/home/muhsin/Desktop/muhsy/.lokiu/PX/plans/WHAT-DONE.md`:

```markdown
## Completed Slice 7: i18n Framework & Error Classification

Implemented comprehensive multi-language support and intelligent error classification.

### Backend Changes
- Added `app/core/i18n_keys.py` with all translatable message keys
- Created `app/modules/catalog/job_errors.py` with error classification system
- Updated `CatalogVariantJob` model with error tracking fields
- Updated `service.py` to raise `JobError` exceptions
- Updated `router.py` to return i18n keys in error responses

### Frontend Changes
- Added i18next configuration and translation files
- Updated components to use `useTranslation()` hook
- Created error handler with i18n key translation
- Added locale detection and switching

### Worker Changes
- Implemented exponential backoff retry logic (2s, 5s, 10s)
- Added error classification for retry decisions
- Implemented graceful shutdown with SIGTERM handling
- Added comprehensive retry logging

### Testing
- ✅ 214+ backend tests passing
- ✅ 291+ frontend tests passing
- ✅ Error classification tests added
- ✅ Worker retry tests added
- ✅ i18n tests passing

### Files Modified/Created
- Created: `app/core/i18n_keys.py`
- Created: `app/modules/catalog/job_errors.py`
- Created: `lib/i18n/index.ts`
- Created: `public/locales/{en,es,fr,de}/translations.json`
- Modified: `app/modules/catalog/models.py`
- Modified: `app/modules/catalog/worker.py`
- Modified: `app/modules/catalog/service.py`
- Modified: `app/modules/catalog/router.py`
- Modified: `app/exceptions/errors.py`
- Modified: `package.json` (added i18next)

### Production Readiness Improvements
- i18n: 100% of error messages now translatable
- Error handling: 100% of errors classified
- Worker resilience: Retryable errors now retry with backoff
- Observability: Error types tracked for monitoring
```

---

## ✅ Final Checklist

- [ ] All BLOCKER items completed
- [ ] All tests passing (backend + frontend)
- [ ] All linting passing (ruff + eslint)
- [ ] All type checks passing (mypy + tsc)
- [ ] No security warnings
- [ ] i18n fully integrated
- [ ] Error classification fully integrated
- [ ] Worker retry logic fully integrated
- [ ] Performance optimizations applied
- [ ] Observability metrics added
- [ ] WHAT-DONE.md updated
- [ ] Code reviewed
- [ ] Ready for production

---

## 🎉 You're Done When

✅ All items above are checked  
✅ All tests pass locally  
✅ Code is reviewed and approved  
✅ WHAT-DONE.md is updated  
✅ No outstanding issues  

---

**Last Updated**: 2026-05-16  
**Total Estimated Time**: 12-16 hours for all BLOCKERs  
**In Progress By**: [Your Name]
