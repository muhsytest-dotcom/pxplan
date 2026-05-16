# Implementation Guides Index

**Purpose**: Explain what documentation exists and which file to read when  
**Created**: 2026-05-16  
**For**: Code agents starting implementation work

---

## 📚 Complete Set of Guides

This directory now contains a complete implementation knowledge base. Each file has a specific purpose and audience.

### 1. **CODE_AGENT_QUICKSTART.md** ⭐ START HERE
**Read Time**: 5 minutes  
**Audience**: Anyone starting work  
**Purpose**: Get oriented quickly

**What you get**:
- Architecture context in 2 minutes
- How to read all other guides
- TL;DR implementation flow
- Common mistakes to avoid
- Quick lookup table

**When to read**: First thing before starting

---

### 2. **IMPLEMENTATION_GUIDE.md** (Main Overview)
**Read Time**: 10-15 minutes  
**Audience**: Project leads, code agents  
**Purpose**: Complete overview with priorities

**What you get**:
- Priority matrix (BLOCKER, HIGH, MEDIUM)
- Effort estimates for each task
- Quick start instructions
- All BLOCKER item summaries
- Testing requirements

**When to read**: After quickstart, before picking a task

---

### 3. **I18N_IMPLEMENTATION.md** (Detailed Guide - Blocker 1)
**Read Time**: 30-45 minutes  
**Audience**: Code agent implementing i18n  
**Purpose**: Step-by-step i18n setup

**What you get**:
- Phase-by-phase breakdown
- Backend i18n keys registry creation
- Translation file setup
- Frontend i18n configuration
- Component integration patterns
- Complete code examples
- Testing checklist
- Validation steps

**When to read**: When starting i18n work

**Covers**:
- Creating i18n_keys.py
- Setting up translation files
- Configuring i18next in React
- Updating components
- Testing translations

---

### 4. **ERROR_CLASSIFICATION.md** (Detailed Guide - Blocker 2 & 3)
**Read Time**: 40-60 minutes  
**Audience**: Code agent implementing error handling & worker retry  
**Purpose**: Step-by-step error classification and retry logic

**What you get**:
- Error classification system design
- Job error classes with retry logic
- Database model updates
- Service layer error handling
- Worker retry loop with exponential backoff
- Complete code examples
- Testing patterns
- Retry strategies explained

**When to read**: When starting error handling or worker work

**Covers**:
- Creating job_errors.py module
- Error type classification
- Model migrations
- Service layer updates
- Worker retry implementation
- Graceful shutdown handling

---

### 5. **IMPLEMENTATION_CHECKLIST.md** (Progress Tracker)
**Read Time**: As needed (reference)  
**Audience**: Code agent implementing  
**Purpose**: Track progress through all implementation phases

**What you get**:
- Checkbox for every single step
- Organized by phase
- Testing requirements per phase
- Validation criteria per section
- Manual testing procedures
- Final completion criteria

**When to read**: Print it out and use it to track progress

**Structure**:
- i18n implementation checklist
- Error classification checklist
- Worker retry checklist
- Testing checklist
- Final validation checklist

---

### 6. **WHAT-DONE.md** (Living Document)
**Read Time**: As needed (reference)  
**Audience**: Everyone  
**Purpose**: Track what's been completed

**What's here**:
- Previous work already done
- What this document should look like after your work
- Template for describing completed work

**When to update**: After completing each major phase

---

## 🗺️ How to Use These Files

### Scenario 1: I'm Starting Implementation
1. Read **CODE_AGENT_QUICKSTART.md** (5 min)
2. Read **IMPLEMENTATION_GUIDE.md** (15 min)
3. Pick a BLOCKER task
4. Go to detailed guide for that task (30-60 min read)
5. Follow detailed guide step-by-step
6. Check off items in **IMPLEMENTATION_CHECKLIST.md**
7. Update **WHAT-DONE.md** when done

### Scenario 2: I'm Implementing i18n
1. Read **CODE_AGENT_QUICKSTART.md** (quick context)
2. Read **I18N_IMPLEMENTATION.md** in full (follow step-by-step)
3. Implement each phase
4. Use **IMPLEMENTATION_CHECKLIST.md** to track
5. Run tests at each phase

### Scenario 3: I'm Implementing Error Handling & Worker
1. Read **CODE_AGENT_QUICKSTART.md**
2. Read **ERROR_CLASSIFICATION.md** in full (follow step-by-step)
3. Phase 1: Create error classification system
4. Phase 2: Update database model
5. Phase 3: Update service layer
6. Phase 4: Update worker with retry logic
7. Test and validate each phase

### Scenario 4: I'm Stuck
1. Re-read the relevant section from detailed guide
2. Look for code examples in the guide
3. Check existing similar code in repository
4. Run tests to see exact error
5. Search for patterns in test files

---

## 📋 What Each File Modifies

### i18n Implementation Touches
- `PX-B/app/core/i18n_keys.py` (CREATE)
- `PX-B/app/exceptions/errors.py` (UPDATE)
- `PX-B/app/modules/catalog/router.py` (UPDATE)
- `PX-F/px/lib/i18n/index.ts` (CREATE)
- `PX-F/px/public/locales/*/translations.json` (CREATE x4)
- `PX-F/px/app/components/VariantStructureStudio.tsx` (UPDATE)
- `PX-F/px/package.json` (UPDATE)

### Error Classification Touches
- `PX-B/app/modules/catalog/job_errors.py` (CREATE)
- `PX-B/app/modules/catalog/models.py` (UPDATE)
- `PX-B/app/modules/catalog/service.py` (UPDATE)
- `PX-B/alembic/versions/` (CREATE migration)

### Worker Retry Touches
- `PX-B/app/modules/catalog/worker.py` (UPDATE)
- `PX-B/Makefile` (UPDATE)
- `PX-B/tests/test_worker_retry.py` (CREATE)

---

## ✅ Completion Indicators

### After i18n (Check All)
- [ ] `npm run i18n:check` passes
- [ ] No hardcoded error strings in code
- [ ] Translation files exist for EN, ES, FR, DE
- [ ] All error responses have i18n_key
- [ ] Frontend uses `useTranslation()` hook

### After Error Classification (Check All)
- [ ] All errors inherit from `JobError`
- [ ] Error types correctly classified
- [ ] Model migration applied successfully
- [ ] `error_type` field in database
- [ ] Tests verify classification

### After Worker Retry (Check All)
- [ ] Worker retries transient errors
- [ ] Worker doesn't retry permanent errors
- [ ] Exponential backoff applied
- [ ] Graceful shutdown works
- [ ] Retry logs clear and helpful

### Overall (Check All)
- [ ] All tests pass: `pytest tests/` and `npm test`
- [ ] No linting errors: `ruff check .` and `npm run lint`
- [ ] No type errors: `mypy app/` and `npm run typecheck`
- [ ] WHAT-DONE.md updated
- [ ] Code reviewed for consistency

---

## 🔄 Reading Order Flowchart

```
START
  ↓
Read CODE_AGENT_QUICKSTART.md (5 min)
  ↓
Read IMPLEMENTATION_GUIDE.md (15 min)
  ↓
Choose Task:
  │
  ├─→ i18n? → Read I18N_IMPLEMENTATION.md
  │
  ├─→ Error Handling? → Read ERROR_CLASSIFICATION.md
  │
  └─→ Worker? → Read ERROR_CLASSIFICATION.md (Phase 3)
  ↓
Implement Following Detailed Guide
  ↓
Check Off Items in IMPLEMENTATION_CHECKLIST.md
  ↓
Run Tests (commands in detailed guide)
  ↓
All Tests Pass?
  ├─→ NO → Fix issues, re-read relevant section
  └─→ YES → Move to next phase/task
  ↓
Update WHAT-DONE.md
  ↓
DONE ✅
```

---

## 📞 Quick Reference

### I Need...

**To understand the project**  
→ CODE_AGENT_QUICKSTART.md + IMPLEMENTATION_GUIDE.md

**To implement i18n**  
→ I18N_IMPLEMENTATION.md + IMPLEMENTATION_CHECKLIST.md

**To implement error handling**  
→ ERROR_CLASSIFICATION.md + IMPLEMENTATION_CHECKLIST.md

**To implement worker retry**  
→ ERROR_CLASSIFICATION.md (Phase 3-4) + IMPLEMENTATION_CHECKLIST.md

**To track my progress**  
→ IMPLEMENTATION_CHECKLIST.md

**To see what's been done**  
→ WHAT-DONE.md

**To understand architecture**  
→ CODE_AGENT_QUICKSTART.md (Architecture Context section)

**To avoid mistakes**  
→ CODE_AGENT_QUICKSTART.md (Common Mistakes section)

**To find code patterns**  
→ CODE_AGENT_QUICKSTART.md (Code Patterns section) or detailed guides

**To run tests**  
→ CODE_AGENT_QUICKSTART.md (Testing Commands section)

---

## 🎯 Key Metrics

### Time Estimates
- i18n: 4-5 hours
- Error Classification: 3-4 hours
- Worker Retry: 2-3 hours
- Testing & Validation: 2-3 hours
- **Total**: 12-16 hours

### Files Modified
- Backend: ~8 files (3 created, 5 updated)
- Frontend: ~6 files (3 created, 3 updated)
- Tests: ~3 files created
- Migrations: 1 file created

### Test Coverage
- New tests: 20-30 test cases
- Existing tests: 214+ backend, 291+ frontend
- Target: 0 failures after implementation

---

## 🚀 Getting Started Right Now

1. **Open this directory** in your editor
2. **Read FILE**: `CODE_AGENT_QUICKSTART.md`
3. **Read FILE**: `IMPLEMENTATION_GUIDE.md`
4. **Choose TASK**: Pick a BLOCKER from priority matrix
5. **Open FILE**: Detailed guide for that task
6. **Follow STEPS**: In the detailed guide
7. **Check PROGRESS**: In `IMPLEMENTATION_CHECKLIST.md`
8. **Update DOCS**: In `WHAT-DONE.md` when done

---

## ✨ These Guides Are Special Because

✅ **Complete**: Every step explained  
✅ **Actionable**: Code examples for each step  
✅ **Tested**: Validation commands provided  
✅ **Ordered**: Do things in right sequence  
✅ **Trackable**: Checklist ensures nothing forgotten  
✅ **Learnable**: Teaches patterns & architecture  
✅ **Recoverable**: Known mistakes section helps debugging  

---

## 📞 Documentation Maintenance

If you find:
- **Unclear sections** → Ask for clarification
- **Missing steps** → Add them before moving on
- **Wrong information** → Fix it and update other docs
- **Outdated content** → Update all related files

**Remember**: These guides are living documents. Make them better as you use them!

---

## 🎓 Learning Path

**Progressive Complexity**:
1. CODE_AGENT_QUICKSTART.md ← Start here (easiest)
2. IMPLEMENTATION_GUIDE.md ← Overview
3. I18N_IMPLEMENTATION.md OR ERROR_CLASSIFICATION.md ← Deep dive
4. IMPLEMENTATION_CHECKLIST.md ← Detailed tracking

**Total Reading Time**: ~2-3 hours before coding starts

---

## ✅ Success Criteria

You know you're ready when you can answer:
- [ ] What is i18n and why do we need it?
- [ ] What's the difference between retryable and non-retryable errors?
- [ ] How does the worker decide to retry or fail?
- [ ] What files do I need to create/modify?
- [ ] How do I test my changes?
- [ ] What's the priority order of tasks?

**If you can't answer these** → Re-read CODE_AGENT_QUICKSTART.md

---

**Last Updated**: 2026-05-16  
**For Questions**: Refer to appropriate detailed guide  
**Ready to Start?** Begin with CODE_AGENT_QUICKSTART.md
