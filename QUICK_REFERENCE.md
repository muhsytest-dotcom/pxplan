# CODE AGENT QUICK REFERENCE CARD
## One-Page Implementation Checklist

**Print this. Keep it visible while coding.**

---

## 🚀 FIVE PHASES IN EXECUTION ORDER

### PHASE 1: Error Classification (3-4 hours)
```
1. Create PX-B/app/modules/catalog/job_errors.py
   - JobErrorType enum (SYSTEM, NETWORK, VALIDATION, CONFLICT, CANCELLED, TIMEOUT, UNKNOWN)
   - RETRYABLE_ERRORS set
   - NON_RETRYABLE_ERRORS set
   - JobError base class with: error_type, message, i18n_key, details
   - Specific error classes

2. Create Alembic migration
   $ cd PX-B && alembic revision --autogenerate -m "add_error_classification"
   
3. Add fields to CatalogVariantJob:
   - error_type: VARCHAR
   - retry_count: INTEGER
   - last_error_at: DATETIME
   - last_error_message: VARCHAR

4. Run migration
   $ alembic upgrade head

5. TEST
   $ pytest tests/test_job_error_classification.py -v
```

### PHASE 2: Worker Retry Logic (2-3 hours)
```
1. Create/Update PX-B/app/modules/catalog/worker.py
   - Implement _execute_with_retry()
   - MAX_RETRIES = 3
   - RETRY_DELAYS = [2, 5, 10]
   - Use exponential backoff with jitter

2. Update service.py
   - Modify _mark_catalog_variant_job() to track error_type, retry_count
   - Update execute_catalog_variant_job() to use _execute_with_retry()

3. Test
   $ pytest tests/test_worker_retry_logic.py -v
   $ pytest tests/test_catalog_variant_*.py -v  # ensure existing tests still pass
```

### PHASE 3: Backend i18n Integration (3-4 hours)
```
1. Create PX-B/app/core/i18n_keys.py
   - CatalogVariantErrorKeys class with all error keys
   - CatalogVariantUIKeys class with all UI keys
   - Copy all from I18N_IMPLEMENTATION.md

2. Update PX-B/app/exceptions/errors.py
   - Add i18n_key parameter to ALL error classes
   - Error responses include: {"i18n_key": "...", "context": {...}}

3. Update PX-B/app/modules/catalog/router.py
   - All error responses include i18n_key
   - Add context data for variable substitution

4. Verify PX-F/px/lib/catalog/i18n.ts
   - All new error keys present
   - Translations complete for en, es, fr, de

5. Update translation files
   - public/locales/en/translations.json
   - public/locales/es/translations.json
   - public/locales/fr/translations.json
   - public/locales/de/translations.json

6. Test
   $ pytest tests/test_i18n_integration.py -v
   $ cd ../PX-F/px && npm run i18n:check && npm test
```

### PHASE 4: Test Coverage (2-4 hours)
```
1. Create PX-B/tests/test_job_error_classification.py
2. Create PX-B/tests/test_worker_retry_logic.py
3. Create PX-B/tests/test_i18n_integration.py
4. Create PX-F/px/app/components/__tests__/error-handler.test.tsx

5. Run full suite
   $ cd PX-B && pytest tests/ -v --cov=app --cov-fail-under=80
   $ cd ../PX-F/px && npm test -- --coverage
```

### PHASE 5: Dead Code Audit (2-3 hours)
```
1. Backend
   $ cd PX-B && vulture app/ --min-confidence 80
   - Remove unused imports, functions
   
2. Frontend
   $ cd ../PX-F/px && npm run lint -- --fix
   - Remove unused components, hooks
```

---

## ✅ PRODUCTION READINESS GATES

**BEFORE MERGING, ALL MUST PASS:**

### Code Quality
- [ ] TypeScript strict mode: `npm run typecheck`
- [ ] Python types: `mypy app/ --ignore-missing-imports`
- [ ] Format check: `black --check app/`
- [ ] Lint check: `flake8 app/` & `npm run lint`

### Testing
- [ ] Backend: `pytest tests/ -v`
- [ ] Frontend: `npm test`
- [ ] Coverage: ≥80% on modified files
- [ ] Manual: Test error scenarios (retry, permanent fail, timeout)

### i18n
- [ ] No hardcoded strings in backend
- [ ] No hardcoded strings in frontend
- [ ] `npm run i18n:check` passes
- [ ] All 4 languages complete

### Database
- [ ] Migration created if model changed
- [ ] Tested: `alembic upgrade head && alembic downgrade -1 && alembic upgrade head`
- [ ] Backward compatible

### Documentation
- [ ] WHAT-DONE.md updated
- [ ] Code has docstrings
- [ ] Comments on non-obvious logic

---

## 🧪 MUST-TEST SCENARIOS

### Error Classification
```python
✓ Retryable errors (SYSTEM_ERROR, NETWORK_ERROR) return is_retryable()=True
✓ Non-retryable errors (VALIDATION_ERROR, CONFLICT_ERROR) return False
✓ Specific error types instantiate correctly
✓ Error details captured in DB
```

### Retry Logic
```python
✓ Transient error → retry → succeed
✓ Permanent error → fail fast (no retries)
✓ Max retries exceeded → mark FAILED
✓ Backoff increases: 2s → 5s → 10s
✓ Jitter prevents thundering herd
✓ Job status transitions: RUNNING → RUNNING (retrying) → COMPLETED/FAILED
```

### i18n
```python
✓ Error response includes i18n_key
✓ Context data passed (e.g., count, limit)
✓ All keys defined in i18n_keys.py
✓ No hardcoded strings in error messages
```

---

## ⚠️ COMMON MISTAKES

| Mistake | Wrong | Right |
|---------|-------|-------|
| Skip tests | "I'll test manually" | Write tests first |
| Hardcode strings | `"Too many variants"` | Use i18n key |
| Catch all errors | `except Exception: pass` | Classify specific errors |
| Retry always | Retry validation errors | Only retry transient |
| Ignore types | `def job(x):` | `def job(x: CatalogVariantJob):` |
| Skip migration | Change model only | Create migration first |
| Incomplete i18n | Only English | All 4 languages |
| No logging | Silent failures | Log with request_id |

---

## 📁 FILES TO CREATE

**Backend**
- [ ] `app/modules/catalog/job_errors.py` (NEW)
- [ ] `app/core/i18n_keys.py` (NEW)
- [ ] `alembic/versions/[TIMESTAMP]_add_error_classification.py` (NEW)
- [ ] `tests/test_job_error_classification.py` (NEW)
- [ ] `tests/test_worker_retry_logic.py` (NEW)
- [ ] `tests/test_i18n_integration.py` (NEW)

**Frontend**
- [ ] `app/components/__tests__/error-handler.test.tsx` (NEW)
- [ ] Update `public/locales/en/translations.json`
- [ ] Update `public/locales/es/translations.json`
- [ ] Update `public/locales/fr/translations.json`
- [ ] Update `public/locales/de/translations.json`

---

## 📊 PHASE DEPENDENCIES

```
Phase 1 ✓ Error Classification
  ↓
Phase 2 ✓ Worker Retry Logic
  ↓
Phase 3 ✓ i18n Backend
  ↓
Phase 4 ✓ Test Coverage
  ↓
Phase 5 ✓ Dead Code Audit
  ↓
PRODUCTION READY ✅
```

---

## 🎯 SUCCESS CHECKLIST

**DONE?**
- [ ] Phase 1 validation gate passes
- [ ] Phase 2 validation gate passes
- [ ] Phase 3 validation gate passes
- [ ] Phase 4 validation gate passes
- [ ] Phase 5 validation gate passes
- [ ] WHAT-DONE.md updated
- [ ] No test failures
- [ ] No lint errors
- [ ] No type errors
- [ ] All i18n keys translated
- [ ] Code reviewed
- [ ] Ready to merge ✓

---

## 🔗 READ THESE DOCUMENTS IN ORDER

1. **CODE_AGENT_QUICKSTART.md** (5 min)
2. **IMPLEMENTATION_GUIDE.md** (10 min)
3. **CODE_AGENT_COMPREHENSIVE_GUIDE.md** (THIS - detailed guide)
4. **I18N_IMPLEMENTATION.md** (i18n specifics)
5. **ERROR_CLASSIFICATION.md** (error handling)
6. **variant-domain-contract.md** (business rules)
7. **WHAT-DONE.md** (track progress)

---

## 💡 REMEMBER

- **Production code**: Tests are not optional
- **Internationalization**: Not "nice to have", it's required
- **Error handling**: Classify properly, retry intelligently
- **Validation gates**: They catch bugs before production
- **WHAT-DONE.md**: Update it as you go
- **Ask questions**: Better now than in production

---

**START HERE: Read CODE_AGENT_COMPREHENSIVE_GUIDE.md completely, then start Phase 1.**
