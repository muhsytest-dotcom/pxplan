# ⚠️ DO NOT FORGET SAFETY CHECKLIST
## Critical Items Code Agents Commonly Miss

**Print this. Check it before every commit.**

---

## 🔴 PRODUCTION BLOCKER ITEMS

**These MUST be done. If you skip any, it will fail in production.**

### Error Classification (Phase 1)
- [ ] `JobError` base class created with `error_type` field
- [ ] `is_retryable()` method returns correct boolean
- [ ] `RETRYABLE_ERRORS` set includes only: SYSTEM, NETWORK
- [ ] `NON_RETRYABLE_ERRORS` set includes: VALIDATION, CONFLICT, CANCELLED, UNKNOWN
- [ ] Specific error classes inherit `JobError`, not generic Exception
- [ ] Database migration created AND applied: `alembic upgrade head`
- [ ] `error_type`, `retry_count`, `last_error_at` fields added to `CatalogVariantJob`
- [ ] Indexes added to error_type and last_error_at

### Worker Retry Logic (Phase 2)
- [ ] `_execute_with_retry()` function implemented
- [ ] MAX_RETRIES = 3 (not more, not less)
- [ ] Retry delays: [2, 5, 10] seconds (exponential backoff)
- [ ] Jitter added to prevent thundering herd (±10%)
- [ ] Retryable errors trigger retry, non-retryable fail fast
- [ ] Job status transitions logged at each retry
- [ ] Worker catches ALL exceptions and wraps as JobError
- [ ] Graceful shutdown on SIGTERM/SIGINT

### i18n Backend (Phase 3)
- [ ] `app/core/i18n_keys.py` module created with all error keys
- [ ] Error classes use `i18n_key` parameter (not hardcoded strings)
- [ ] Error responses include: `{"error": {"i18n_key": "...", "context": {...}}}`
- [ ] All router endpoints return i18n_key on errors
- [ ] No user-visible strings hardcoded in backend
- [ ] Frontend can access all i18n keys from backend errors

### Internationalization (Phases 3 & 4)
- [ ] All 4 languages have translations: en, es, fr, de
- [ ] `npm run i18n:check` passes
- [ ] No English placeholder strings in non-English files
- [ ] Error messages translated in ALL languages
- [ ] UI labels translated in ALL languages
- [ ] Variable substitution works (e.g., {count}, {limit})

### Testing (Phase 4)
- [ ] `test_job_error_classification.py` created and passes
- [ ] `test_worker_retry_logic.py` created and passes
- [ ] `test_i18n_integration.py` created and passes
- [ ] Coverage ≥80%: `pytest tests/ --cov=app --cov-fail-under=80`
- [ ] All existing tests still pass
- [ ] Frontend tests pass: `npm test`
- [ ] No test timeouts (increase if needed)

---

## 🟡 CODE QUALITY (MUST PASS)

**These are not optional. Production code requires them.**

### Type Safety
- [ ] Python: `mypy app/ --ignore-missing-imports` passes
- [ ] TypeScript: `npm run typecheck` passes
- [ ] No `type: ignore` comments without explanation
- [ ] All function parameters have type hints
- [ ] All function return types specified

### Code Formatting
- [ ] Python: `black app/` has been run
- [ ] Python: `flake8 app/ --max-line-length=100` passes
- [ ] TypeScript: `npm run lint` passes
- [ ] No inconsistent spacing/indentation
- [ ] Imports organized (isort for Python, ESLint for TS)

### No Hardcoded Values
- [ ] No hardcoded timeouts (use settings)
- [ ] No hardcoded error strings (use i18n keys)
- [ ] No hardcoded limits (use config)
- [ ] No hardcoded API endpoints (use base URL)
- [ ] No hardcoded email addresses or URLs

### Error Handling
- [ ] No bare `except:` statements
- [ ] No `except Exception:` with pass
- [ ] All exceptions caught specifically, classified correctly
- [ ] Errors logged with request_id and context
- [ ] User-facing errors use i18n keys
- [ ] Internal errors logged for debugging

### Security
- [ ] No credentials in logs
- [ ] No sensitive data in error messages
- [ ] User input validated before DB operations
- [ ] Rate limiting enforced on mutation endpoints
- [ ] Store ownership verified on all endpoints
- [ ] SQL injection prevented (use ORM, parameterized queries)

---

## 🧪 TESTING (DO NOT SKIP)

**Manual testing is NOT sufficient. Automated tests catch edge cases.**

### Error Classification Tests
- [ ] Test `is_retryable()` for each error type
- [ ] Test specific error instantiation
- [ ] Test error details captured (message, i18n_key, context)
- [ ] Test error_type stored in database
- [ ] Test JobError subclasses work correctly

### Retry Logic Tests
- [ ] Test transient error → retry → succeed path
- [ ] Test permanent error fails immediately (no retries)
- [ ] Test max retries exceeded → FAILED status
- [ ] Test backoff delays increase correctly
- [ ] Test jitter is applied to delays
- [ ] Test job status transitions through retries
- [ ] Test concurrent retries don't interfere
- [ ] Test retry_count increments correctly

### i18n Tests
- [ ] Test error response includes i18n_key
- [ ] Test context data passed for substitution
- [ ] Test all i18n keys are defined
- [ ] Test missing keys handled gracefully
- [ ] Test all 4 languages have translations
- [ ] Test variable substitution works (plural forms, numbers)

### Integration Tests
- [ ] Test error → frontend translation → display
- [ ] Test retry causes eventual success
- [ ] Test cancelled jobs clean up properly
- [ ] Test timeout detection works
- [ ] Test concurrent jobs don't conflict

---

## 📋 DATABASE (DO NOT SKIP)

**Wrong database changes = data corruption or downtime.**

### Migrations
- [ ] Alembic migration created: `alembic revision --autogenerate`
- [ ] Migration tested locally: `alembic upgrade head`
- [ ] Migration rollback tested: `alembic downgrade -1`
- [ ] Migration is backward compatible
- [ ] No data loss when rolling back
- [ ] Indexes added for performance-critical columns
- [ ] Column constraints correct (nullable, defaults, etc.)

### Model Updates
- [ ] SQLModel/SQLAlchemy model updated to match migration
- [ ] Field types match database column types
- [ ] Optional fields marked as `| None`
- [ ] Required fields properly enforced
- [ ] Relationships properly defined
- [ ] Unique constraints enforced in model

### Data Integrity
- [ ] No orphaned records created
- [ ] Foreign key relationships maintained
- [ ] Transactions are atomic (all-or-nothing)
- [ ] Concurrent updates handled correctly
- [ ] No race conditions in job status updates

---

## 📦 DEPENDENCIES (DO NOT SKIP)

**Missing dependencies = ImportError in production.**

### Backend Dependencies
- [ ] All imports in code exist in requirements.txt
- [ ] No version conflicts in requirements.txt
- [ ] Run: `pip install -r requirements.txt` successfully
- [ ] Can import all modules without errors

### Frontend Dependencies
- [ ] All npm imports exist in package.json
- [ ] No version conflicts in package.json
- [ ] Run: `npm install` successfully
- [ ] Can import all modules without errors
- [ ] No deprecated packages

### No Circular Imports
- [ ] Python: No circular import chains
- [ ] TypeScript: No circular import chains
- [ ] All imports resolvable

---

## 🔐 SECURITY (DO NOT SKIP)

**Security issues are not optional.**

### Credentials & Secrets
- [ ] No API keys in code
- [ ] No passwords in code
- [ ] No tokens in version control
- [ ] Credentials loaded from environment variables
- [ ] Never log sensitive data
- [ ] Error messages don't expose internal details

### Input Validation
- [ ] User input validated before using in queries
- [ ] Request payloads validated against schemas
- [ ] File uploads checked for type/size
- [ ] Rate limiting enforced
- [ ] CSRF tokens checked on mutations

### Access Control
- [ ] Store ownership verified
- [ ] User permissions checked
- [ ] Tenant isolation enforced
- [ ] Admin-only endpoints protected
- [ ] User can't see other users' data

---

## 📊 OBSERVABILITY (DO NOT SKIP)

**If you can't see what's happening, you can't debug production.**

### Logging
- [ ] All errors logged with request_id
- [ ] Job state transitions logged
- [ ] Retry attempts logged (with error type)
- [ ] Success/failure logged
- [ ] Performance metrics logged (duration, queue wait)
- [ ] Structured logging (JSON preferred)

### Metrics
- [ ] Job count by status tracked
- [ ] Error count by type tracked
- [ ] Processing latency measured
- [ ] Queue depth monitored
- [ ] Success rate calculated
- [ ] Metrics endpoint available

### Tracing
- [ ] Request ID propagated through all layers
- [ ] Trace spans for long-running operations
- [ ] Error context available in traces

---

## 📝 DOCUMENTATION (DO NOT SKIP)

**Future you will forget. Document now.**

### Code Comments
- [ ] Docstrings on all public functions
- [ ] Comments on non-obvious logic
- [ ] Comments on business rule implementations
- [ ] Comments on error handling choices

### README Updates
- [ ] New setup steps documented
- [ ] New environment variables documented
- [ ] New database columns documented
- [ ] Migration instructions included

### WHAT-DONE.md
- [ ] Updated with completed work
- [ ] Includes what was changed
- [ ] Notes any gotchas or decisions

---

## 🚀 DEPLOYMENT CHECKLIST (DO NOT SKIP)

**Can't go to production without these.**

### Build Verification
- [ ] Backend builds: `pytest tests/ -v`
- [ ] Frontend builds: `npm run build`
- [ ] No build warnings
- [ ] No type errors in build output

### Performance
- [ ] Database queries optimized (no N+1)
- [ ] Retry backoff doesn't exceed 30 seconds
- [ ] Job cancellation responsive (<5 seconds)
- [ ] No memory leaks in retry loop

### Configuration
- [ ] All environment variables documented
- [ ] Default values are secure (not DEBUG=True)
- [ ] Configuration works for: dev, staging, prod
- [ ] Secrets not in config files

### Scalability
- [ ] Worker can handle max queue depth
- [ ] Database can handle concurrent jobs
- [ ] Memory usage stays reasonable
- [ ] No thread/connection pool exhaustion

---

## 🎯 BEFORE EVERY COMMIT

**Copy-paste this and check each item:**

```
PRODUCTION BLOCKER (Phase 1-4)
- [ ] Error classification complete
- [ ] Worker retry logic complete
- [ ] i18n backend complete
- [ ] Tests passing (80%+ coverage)
- [ ] All 4 languages have translations

CODE QUALITY
- [ ] mypy passes
- [ ] eslint/black passes
- [ ] flake8 passes
- [ ] npm typecheck passes

TESTING
- [ ] Backend tests: pytest tests/ -v
- [ ] Frontend tests: npm test
- [ ] Manual test: error → retry → succeed
- [ ] Manual test: permanent error → fail fast

DATABASE
- [ ] Migration created (if model changed)
- [ ] Migration tested: upgrade && downgrade && upgrade
- [ ] No data corruption

SECURITY
- [ ] No credentials in code
- [ ] User input validated
- [ ] Store ownership checked
- [ ] Rate limiting enforced

OBSERVABILITY
- [ ] Errors logged with request_id
- [ ] Retry attempts logged
- [ ] Metrics available

DOCUMENTATION
- [ ] WHAT-DONE.md updated
- [ ] Code has comments/docstrings
- [ ] Gotchas documented

READY?
- [ ] All above checked ✓
```

---

## 🚨 IF YOU'RE UNSURE

**Do NOT skip to make it "faster":**

1. ❌ Skip tests → Hidden bugs, production failures
2. ❌ Hardcode strings → Non-English users see gibberish
3. ❌ Skip migration → Database corruption
4. ❌ Retry everything → Permanent errors retry forever
5. ❌ No logging → Can't debug production issues
6. ❌ Skip type hints → Silent runtime errors
7. ❌ Hardcode values → Can't configure for different environments

**When in doubt:**
- Re-read CODE_AGENT_COMPREHENSIVE_GUIDE.md
- Check variant-domain-contract.md for business rules
- Check existing code for patterns
- Write a test to verify behavior
- Run validation gates

---

## ✅ FINAL CHECK BEFORE MERGE

**Create this file before committing (copy-paste and fill in):**

```markdown
# Implementation Verification

**Date**: YYYY-MM-DD  
**Implementer**: Your Name  
**Phase**: 1 / 2 / 3 / 4 / 5  

## Code Quality
- [ ] All tests pass: ✓
- [ ] Coverage ≥80%: ✓
- [ ] Type checks pass: ✓
- [ ] Linting passes: ✓

## Functional Verification
- [ ] Happy path works: ✓
- [ ] Error path works: ✓
- [ ] Retry path works: ✓
- [ ] i18n works for all languages: ✓

## Database
- [ ] Migration applied: ✓
- [ ] Rollback tested: ✓
- [ ] No data corruption: ✓

## Security
- [ ] No credentials exposed: ✓
- [ ] User isolation enforced: ✓
- [ ] Input validation present: ✓

## Observability
- [ ] Logging complete: ✓
- [ ] Metrics available: ✓
- [ ] Traces working: ✓

## Documentation
- [ ] WHAT-DONE.md updated: ✓
- [ ] Code commented: ✓
- [ ] README updated (if needed): ✓

## Ready for Merge?
YES ✓ / NO ✗

If NO, complete items above before committing.
```

---

## 📞 STILL UNSURE?

Read in order:
1. **QUICK_REFERENCE.md** (1 min)
2. **CODE_AGENT_COMPREHENSIVE_GUIDE.md** relevant phase (5-10 min)
3. **variant-domain-contract.md** (business rules)

Still stuck? Check:
- Existing tests for patterns
- Existing code for implementations
- Git history for similar changes

---

**Don't skip the checklist. It catches real bugs.**

**Print this page. Check every item before commit.**

**Production code requires discipline.**
