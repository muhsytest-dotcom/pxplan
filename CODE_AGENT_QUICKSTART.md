# Code Agent Quick Start Reference

**Purpose**: Get oriented quickly before starting implementation  
**Read Time**: 5 minutes  
**Next Step**: Pick a task from priority matrix and go to detailed guide

---

## 🎯 What You're Doing

You're fixing critical gaps in a production ecommerce variants feature (PX). The system generates product variants (e.g., a shirt in all size/color combinations) with async jobs, SSE streaming, and snapshot-based isolation.

**Current Status**: 65-70% complete, but 3 critical gaps blocking production.

---

## 📂 File Organization

All guidance files are in `/home/muhsin/Desktop/muhsy/.lokiu/PX/plans/`:

| File | Purpose | Read When |
|------|---------|-----------|
| **IMPLEMENTATION_GUIDE.md** | Priorities, phases, overview | First - get overview |
| **I18N_IMPLEMENTATION.md** | i18n setup step-by-step | Starting i18n work |
| **ERROR_CLASSIFICATION.md** | Error handling/retry | Starting worker/error work |
| **WHAT-DONE.md** | Track completed work | After finishing each phase |

---

## 🚀 Start Here (TL;DR)

### Step 1: Read This Entire Section (2 min)

### Step 2: Read `IMPLEMENTATION_GUIDE.md` (3 min)

### Step 3: Pick ONE BLOCKER Task
- [ ] i18n Framework
- [ ] Fix Worker Exception Handling
- [ ] Add Error Classification

### Step 4: Go to Detailed Guide
- i18n → `I18N_IMPLEMENTATION.md`
- Error handling → `ERROR_CLASSIFICATION.md`

### Step 5: Implement Following That Guide

### Step 6: Test Using Commands in Guide

### Step 7: Commit & Mark Done in `WHAT-DONE.md`

---

## 🏗️ Architecture Context

### Backend (PX-B)
- FastAPI server with async jobs
- SQLAlchemy ORM for DB
- Worker process for variant generation
- Event bus for SSE streaming

### Frontend (PX-F)
- Next.js + React 19
- No i18n yet (need to add)
- EventSource for SSE
- Context-based state management

### Key Pattern: Immutable Snapshots
- When job starts: capture product structure snapshot
- Job works against snapshot, not live data
- On success: atomically swap published snapshot
- Storefront only sees published snapshots

---

## 📋 Blockers Explained Simply

### Blocker 1: i18n Missing
**Problem**: System supports multiple languages but has 0 i18n.  
**Fix**: Add translation framework + string keys.  
**Impact**: Users can't see messages in their language.

### Blocker 2: Worker Exception Handling Bad
**Problem**: All exceptions caught the same way, no retry logic.  
**Fix**: Classify errors, retry transient ones.  
**Impact**: Temporary DB hiccups cause permanent failures.

### Blocker 3: Error Classification Missing
**Problem**: Can't tell if error is permanent or transient.  
**Fix**: Add error_type field to model, classify errors.  
**Impact**: No way to decide whether to retry.

---

## 🎬 Implementation Flow

```
┌─────────────────────────────────────┐
│ 1. i18n Framework (4-5 hours)      │
│    - Create i18n keys              │
│    - Update error responses        │
│    - Add translations              │
│    - Wire up frontend              │
└──────────────┬──────────────────────┘
               ↓
┌─────────────────────────────────────┐
│ 2. Error Classification (3-4 hrs)  │
│    - Create JobError classes       │
│    - Add model fields              │
│    - Update service layer          │
└──────────────┬──────────────────────┘
               ↓
┌─────────────────────────────────────┐
│ 3. Worker Retry Logic (2-3 hrs)    │
│    - Update worker loop            │
│    - Add exponential backoff       │
│    - Handle graceful shutdown      │
└──────────────┬──────────────────────┘
               ↓
┌─────────────────────────────────────┐
│ 4. Run All Tests                    │
│    - pytest tests/                  │
│    - npm test                       │
│    - Lint + type check              │
└──────────────┬──────────────────────┘
               ↓
┌─────────────────────────────────────┐
│ 5. Update WHAT-DONE.md              │
│    Document everything completed    │
└─────────────────────────────────────┘
```

---

## 🧪 Testing Commands (Use These!)

### Backend
```bash
cd PX-B

# Test one module
pytest tests/test_catalog_variant_ops.py -v

# Test with coverage
pytest tests/ -v --cov=app --cov-report=term-missing

# Specific test
pytest tests/test_job_error_classification.py::TestErrorClassification -v

# Type checking
mypy app/

# Linting
ruff check .
```

### Frontend
```bash
cd PX-F/px

# Run tests
npm test

# Check i18n
npm run i18n:check

# Type checking
npm run typecheck

# Linting
npm run lint
```

### Validation (Must Pass All!)
```bash
# Backend: MUST PASS
cd PX-B && python -m pytest tests/ -v && ruff check . && mypy app/

# Frontend: MUST PASS
cd PX-F/px && npm test && npm run typecheck && npm run lint
```

---

## 📍 Key Files You'll Edit

### Backend
| File | What to Edit | Why |
|------|--------------|-----|
| `app/core/i18n_keys.py` | **CREATE** new file | Central error key registry |
| `app/modules/catalog/job_errors.py` | **CREATE** new file | Error classification |
| `app/modules/catalog/models.py` | Add error fields to `CatalogVariantJob` | Track error type & retry count |
| `app/modules/catalog/worker.py` | Update retry loop | Smart retry logic |
| `app/modules/catalog/service.py` | Update error handling | Raise `JobError` not generic |
| `app/modules/catalog/router.py` | Return i18n keys | Frontend can translate |
| `app/exceptions/errors.py` | Add `message_key` | All errors have keys |

### Frontend
| File | What to Edit | Why |
|------|--------------|-----|
| `package.json` | Add `i18next` | i18n framework |
| `lib/i18n/index.ts` | **CREATE** new file | i18n config |
| `public/locales/en/translations.json` | **CREATE** new file | English translations |
| `app/components/VariantStructureStudio.tsx` | Update to use `useTranslation()` | Show translated errors |
| `lib/catalog/error-handler.ts` | Update error display | Translate error keys |

---

## 🔍 Code Patterns to Follow

### Error Pattern (Backend)
```python
# DON'T DO THIS (old way)
try:
    do_something()
except Exception as e:
    logger.error(f"Error: {e}")
    raise

# DO THIS (new way)
from app.modules.catalog.job_errors import JobError, JobErrorType

try:
    do_something()
except SpecificError as e:
    raise JobError(
        error_type=JobErrorType.VALIDATION_ERROR,
        message="What happened",
        i18n_key="catalog.variant.errors.specific_error",
        details={"context": "value"}
    )
```

### Translation Pattern (Frontend)
```typescript
// DON'T DO THIS (old way)
<button>{jobStatus.error}</button>

// DO THIS (new way)
import { useTranslation } from 'react-i18next';

function ErrorDisplay() {
  const { t } = useTranslation();
  return (
    <button>{t(jobStatus.errorKey)}</button>
  );
}
```

---

## ✅ Definition of Done

For EACH task, mark complete only when:
- [ ] Code written per detailed guide
- [ ] All tests pass (pytest + npm test)
- [ ] Linting passes (ruff + eslint)
- [ ] Type checking passes (mypy + tsc)
- [ ] Changes documented in WHAT-DONE.md
- [ ] Code reviewed for consistency with existing code
- [ ] No breaking changes to API contracts

---

## 🆘 If You Get Stuck

1. **Re-read the detailed guide** for that task
2. **Check existing similar code** - patterns already exist
3. **Run tests** to see exact failures
4. **Look at git history** - similar changes done before
5. **Ask for clarification** - unclear requirements

---

## 📊 Progress Tracking

Update `/home/muhsin/Desktop/muhsy/.lokiu/PX/plans/WHAT-DONE.md` as you complete:

```markdown
## Completed Slice [X]: [Task Name]

Status: ✅ Complete / 🟡 In Progress

### Changes Made
- List of what was changed
- Files modified/created
- Tests added

### Tests Passing
- ✅ Backend tests: X passed
- ✅ Frontend tests: Y passed

### Known Issues
- None / List any issues

### Next Steps
- What comes after this
```

---

## 🎓 Learning Resources in Codebase

Look at these to understand existing patterns:

### Error Handling Pattern
- `PX-B/app/exceptions/errors.py` - Existing error classes
- `PX-B/tests/test_catalog_variant_ops.py` - How tests expect errors

### i18n Pattern (Backend)
- `PX-B/app/core/i18n.py` - Existing locale handling
- `PX-B/tests/test_config_validation.py` - Config i18n patterns

### Async Job Pattern
- `PX-B/app/modules/catalog/service.py` - Job execution (~line 1813)
- `PX-B/app/modules/catalog/worker.py` - Existing worker

### Frontend Pattern
- `PX-F/px/app/components/VariantStructureStudio.tsx` - Context usage
- `PX-F/px/lib/catalog/variant-job-events.ts` - SSE pattern

---

## 🚨 Common Mistakes (Avoid These)

| Mistake | Why Bad | Fix |
|---------|---------|-----|
| Hardcoding error messages | Can't translate | Use i18n keys |
| Catching generic `Exception` | Can't classify errors | Use specific `JobError` |
| Retry all errors | Some never succeed | Only retry transient ones |
| Not testing changes | Breaks other things | Run full test suite |
| Mixing concerns | Harder to maintain | Keep errors/i18n/retry separate |

---

## 📞 Quick Lookup

**Q: Where are the database models?**  
A: `PX-B/app/modules/catalog/models.py`

**Q: Where's the API code?**  
A: `PX-B/app/modules/catalog/router.py`

**Q: Where are the business logic functions?**  
A: `PX-B/app/modules/catalog/service.py`

**Q: Where's the async worker?**  
A: `PX-B/app/modules/catalog/worker.py`

**Q: Where are tests?**  
A: `PX-B/tests/test_catalog_*.py` (backend), `PX-F/px/test/` (frontend)

**Q: How do I run tests?**  
A: Backend: `cd PX-B && pytest tests/`. Frontend: `cd PX-F/px && npm test`

**Q: How do I start development?**  
A: Backend: `make dev`. Frontend: `cd PX-F/px && npm run dev`

---

## 🏁 You're Ready!

1. ✅ You understand what needs fixing
2. ✅ You know the priority order
3. ✅ You have detailed guides for each task
4. ✅ You know test & validation commands
5. ✅ You know common patterns & mistakes

**Next**: Pick a BLOCKER task → Go to detailed guide → Implement!

---

**Last Updated**: 2026-05-16  
**Questions?** Look at the detailed guide for your task or existing code patterns.
