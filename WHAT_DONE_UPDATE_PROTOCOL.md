# ⚠️ WHAT-DONE.md UPDATE PROTOCOL
## Critical Timing Instructions

**IMPORTANT**: This document explains when and how to update WHAT-DONE.md

---

## 🔴 CRITICAL RULE

### **UPDATE WHAT-DONE.md ONLY AFTER IMPLEMENTATION IS FULLY COMPLETED**

**DO NOT UPDATE IT BEFORE IMPLEMENTATION IS COMPLETE**

---

## 📋 WHAT-DONE.md UPDATE TIMING

### ❌ DO NOT UPDATE WHAT-DONE.md:
- While planning ("I'm about to implement...")
- Partially complete ("I finished half of...")
- During implementation ("I created job_errors.py...")
- Before testing ("I wrote the code but haven't tested...")
- Before validation gates ("Tests don't pass yet but...")
- With incomplete information

### ✅ ONLY UPDATE WHAT-DONE.md:
- **AFTER** implementation is fully complete
- **AFTER** all validation gates pass
- **AFTER** tests pass (≥80% coverage)
- **AFTER** code review/linting complete
- **AFTER** commit is ready
- **WITH** complete information about what was done

---

## 🎯 EXACT PROTOCOL

### Phase Implementation Cycle

**Step 1: Read & Plan** (no changes to WHAT-DONE.md)
```
✓ Read CODE_AGENT_COMPREHENSIVE_GUIDE.md for your phase
✓ Read detailed spec guide (ERROR_CLASSIFICATION.md, etc.)
✓ Understand requirements
✗ DO NOT UPDATE WHAT-DONE.md yet
```

**Step 2: Implement** (no changes to WHAT-DONE.md)
```
✓ Create required files
✓ Write code following patterns
✓ Write tests
✓ Run tests repeatedly as you develop
✗ DO NOT UPDATE WHAT-DONE.md yet
```

**Step 3: Validate & Test** (no changes to WHAT-DONE.md)
```
✓ Run validation gates: pytest tests/test_job_error_classification.py -v
✓ Run full test suite: pytest tests/ -v --cov=app
✓ Check types: mypy app/
✓ Check linting: black --check app/ && flake8 app/
✓ Verify coverage ≥80%
✗ DO NOT UPDATE WHAT-DONE.md yet
```

**Step 4: Check Quality** (no changes to WHAT-DONE.md)
```
✓ Review your code for production quality
✓ Check DO_NOT_FORGET.md - all items done?
✓ Verify all validation gates pass
✓ Run git diff to review changes
✗ DO NOT UPDATE WHAT-DONE.md yet
```

**Step 5: Document & Commit** (NOW update WHAT-DONE.md)
```
✓ All tests pass ✓
✓ All validation gates pass ✓
✓ Code quality complete ✓
✓ Coverage ≥80% ✓
NOW: Update WHAT-DONE.md with what you accomplished
✓ Commit: git commit -am "Phase 1: Error Classification Complete"
```

---

## 📝 WHAT TO WRITE IN WHAT-DONE.md

### Section Format

When you update WHAT-DONE.md, add a new "Completed Slice" section:

```markdown
## Completed Slice [N]: [Phase Name]

Implemented [feature/phase description].

**Files Created:**
- `app/modules/catalog/job_errors.py` - JobErrorType enum, JobError base class
- `alembic/versions/[TIMESTAMP]_add_error_classification.py` - DB migration
- `tests/test_job_error_classification.py` - Error classification tests

**Files Modified:**
- `app/modules/catalog/models.py` - Added error_type, retry_count fields
- `app/modules/catalog/service.py` - Updated _mark_catalog_variant_job()

**Key Changes:**
- Created JobErrorType enum with RETRYABLE and NON_RETRYABLE classification
- Added error tracking to CatalogVariantJob model
- Implemented is_retryable() method for error classification
- Added database migration (applied successfully)
- Created comprehensive test suite

**Tests:**
- Created `test_job_error_classification.py` (15 test cases)
- All tests passing: ✓
- Coverage: 95% on catalog module

**Validation:**
- ✓ pytest tests/test_job_error_classification.py -v (all pass)
- ✓ pytest tests/ -v --cov=app --cov-fail-under=80 (coverage ≥80%)
- ✓ mypy app/ (no type errors)
- ✓ black --check app/ (formatted)
- ✓ flake8 app/ (no lint errors)

**Notes:**
- [Any important decisions or gotchas]
- [Any follow-up work needed]
- [Any changes to architecture or approach]
```

---

## ⏰ TIMING EXAMPLES

### Example 1: Phase 1 Implementation

**Timeline:**
- 14:00 - Read guides (no WHAT-DONE.md changes)
- 15:00 - Create job_errors.py (no WHAT-DONE.md changes)
- 15:30 - Create migration (no WHAT-DONE.md changes)
- 16:00 - Write tests (no WHAT-DONE.md changes)
- 16:30 - All tests pass, validation gates pass (no WHAT-DONE.md changes yet)
- 17:00 - Final code review, check DO_NOT_FORGET.md (no WHAT-DONE.md changes yet)
- 17:15 - **NOW UPDATE WHAT-DONE.md** ✓
- 17:20 - Commit to git

### Example 2: Multi-Phase Handoff

**Timeline:**
- Phase 1 complete ✓ → Update WHAT-DONE.md ✓ → Commit ✓
- Phase 2 complete ✓ → Update WHAT-DONE.md ✓ → Commit ✓
- Phase 3 complete ✓ → Update WHAT-DONE.md ✓ → Commit ✓
- All phases complete ✓ → WHAT-DONE.md fully updated ✓ → Final commit ✓

---

## 🔍 HOW TO VERIFY YOU'RE READY TO UPDATE WHAT-DONE.md

**Checklist - ALL must be ✓ before updating WHAT-DONE.md:**

- [ ] Code implemented (all files created/modified)
- [ ] Tests written (following test patterns)
- [ ] Validation gates passing (pytest tests/test_*.py -v)
- [ ] Full test suite passing (pytest tests/ -v)
- [ ] Coverage ≥80% (pytest --cov=app --cov-fail-under=80)
- [ ] Type checking passing (mypy app/)
- [ ] Code formatted (black app/)
- [ ] Linting passing (flake8 app/)
- [ ] DO_NOT_FORGET.md checklist completed (115+ items)
- [ ] Code reviewed (for production quality)
- [ ] No outstanding issues or TODOs
- [ ] Ready to commit to git

**If ANY checkbox is unchecked: DO NOT UPDATE WHAT-DONE.md YET**

Once all are checked: You're ready to update WHAT-DONE.md

---

## 📌 REMINDER FOR CODE AGENTS

When you see this instruction in AGENT_PROMPT_TEMPLATE.md:

```
✅ Update WHAT-DONE.md after phase completion (not before!)
```

It means:
1. Complete ALL work for the phase
2. Run ALL validation gates
3. Verify ALL tests pass
4. Check DO_NOT_FORGET.md
5. Then and ONLY THEN update WHAT-DONE.md
6. Then commit

**DO NOT update WHAT-DONE.md before all above are complete.**

---

## 🎯 WHY THIS PROTOCOL MATTERS

### Problem: Early WHAT-DONE.md Updates
- ❌ Creates false sense of completion
- ❌ Confuses next agent ("It says done, but tests fail")
- ❌ Makes progress tracking inaccurate
- ❌ Prevents proper rollback if issues found later

### Solution: Update ONLY When Truly Complete
- ✅ Accurate progress tracking
- ✅ Next agent knows what's actually done
- ✅ Can use WHAT-DONE.md as source of truth
- ✅ Proper continuity between agents

---

## ✅ FINAL CHECKLIST

**Before you finish a phase:**

```
FINAL VERIFICATION (do this before updating WHAT-DONE.md):

Code Quality:
  [ ] All tests pass
  [ ] Coverage ≥80%
  [ ] No type errors
  [ ] No linting errors
  
Validation:
  [ ] Phase validation gate passes
  [ ] Full test suite passes
  [ ] All DO_NOT_FORGET.md items checked
  
Documentation:
  [ ] Ready to describe phase in WHAT-DONE.md
  [ ] Know what files were created/modified
  [ ] Know what was tested
  [ ] Know what validation passed
  
Then:
  [ ] Update WHAT-DONE.md
  [ ] Commit with clear message
  [ ] Move to next phase
```

If all checked, you're done with the phase.

**Then** you update WHAT-DONE.md and commit.

---

**Remember: WHAT-DONE.md is updated AFTER implementation is complete, not before.**
