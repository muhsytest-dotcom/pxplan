# 🤖 STANDARD PROMPT FOR CODE AGENTS
## Use This When Handoff Work to Code Agents

---

## 📋 COPY-PASTE THIS PROMPT WHEN BRIEFING CODE AGENTS

```
===============================================================================
TASK: Implement Production-Grade Variants Feature - Error Classification Phase
===============================================================================

PROJECT CONTEXT:
- You're working on PX, a multi-language e-commerce platform
- Current status: 65-70% complete, 3 critical blockers remaining
- This work is production-critical and must follow strict quality standards
- Tests are NOT optional - they're validation gates
- All code must pass: tests, linting, type checking, and code review

YOUR TASK:
Implement Phase 1: Error Classification System (3-4 hours)

CRITICAL REQUIREMENTS (DO NOT SKIP):
✅ Create JobErrorType enum with error classification
✅ Add error_type field to CatalogVariantJob database model
✅ Create job_errors.py module with JobError base class
✅ Create Alembic database migration
✅ Write comprehensive tests (test patterns provided)
✅ Pass validation gate: pytest tests/test_job_error_classification.py -v
✅ Update WHAT-DONE.md after phase completion (not before!)
✅ Check DO_NOT_FORGET.md before every commit (115+ items)

DELIVERABLES:
- JobError classification system ✓
- Database migration (applied) ✓
- Test coverage ≥80% ✓
- All validation gates passing ✓
- WHAT-DONE.md updated ✓

CONSTRAINT: This is production code. No shortcuts on testing or quality.

DOCUMENTS TO READ (in order):
1. AGENT_HANDOFF.md (this handoff document) - 10 min
2. 00_START_HERE.md - 5 min (understand guide system)
3. CODE_AGENT_COMPREHENSIVE_GUIDE.md PHASE 1 section - 20 min
4. ERROR_CLASSIFICATION.md - 10 min (detailed spec)
5. Print QUICK_REFERENCE.md (keep visible while coding)

NEXT STEPS:
1. Read all documents above
2. Follow CODE_AGENT_COMPREHENSIVE_GUIDE.md PHASE 1 step-by-step
3. Create required files: job_errors.py, alembic migration
4. Write tests following test patterns provided
5. Run validation gates
6. Check DO_NOT_FORGET.md before committing
7. Update WHAT-DONE.md after completion
8. Commit with clear message documenting changes

WORKSPACE:
- Backend: d:\Github\muhsinmuhsy\PX\PX-B
- Frontend: d:\Github\muhsinmuhsy\PX\PX-F
- Plans: d:\Github\muhsinmuhsy\PX\pxplan
- Read all .md files from pxplan folder

COMMANDS YOU'LL RUN:
# Create migration (after reading guide)
cd PX-B && alembic revision --autogenerate -m "add_error_classification_to_jobs"

# Apply migration
alembic upgrade head

# Run validation gate (must pass)
pytest tests/test_job_error_classification.py -v

# Full test suite (must pass)
pytest tests/ -v

# Type check
mypy app/ --ignore-missing-imports

# Format check
black --check app/

# Lint check
flake8 app/ --max-line-length=100

SUCCESS CRITERIA:
✅ All validation gates pass
✅ Test coverage ≥80%
✅ Zero type errors
✅ Zero lint errors
✅ WHAT-DONE.md updated
✅ Code reviewed and approved

IMPORTANT NOTES:
- Don't skip tests - they catch real bugs
- Don't hardcode strings - use i18n keys
- Don't skip database migration - data integrity depends on it
- Don't deploy without passing all validation gates
- Check DO_NOT_FORGET.md before EVERY commit

Questions? Read the relevant .md file from pxplan folder.
The answer is almost certainly there.

Good luck! 🚀
===============================================================================
```

---

## 📝 VARIATION: FOR MULTIPLE PHASES

### If Handing Off 2+ Phases

```
===============================================================================
TASK: Implement Production-Grade Variants Feature - Phases 1-3
===============================================================================

CONTEXT:
- 5 critical phases to implement (14-18 hours total)
- Must be done in sequential order (Phase 2 depends on Phase 1, etc.)
- This is production code - quality is non-negotiable

PHASES (in order):
1. Phase 1: Error Classification (3-4 hours)
   - CREATE: job_errors.py with JobErrorType enum
   - FILES: models.py, alembic migration
   - VALIDATION: pytest tests/test_job_error_classification.py -v

2. Phase 2: Worker Retry Logic (2-3 hours)
   - CREATE: Retry logic in worker.py with exponential backoff
   - FILES: service.py modifications
   - VALIDATION: pytest tests/test_worker_retry_logic.py -v

3. Phase 3: Backend i18n Integration (3-4 hours)
   - CREATE: app/core/i18n_keys.py with all error keys
   - FILES: exceptions/errors.py, router.py modifications
   - VALIDATION: pytest tests/test_i18n_integration.py -v && npm run i18n:check

CRITICAL:
- Complete Phase 1 fully before starting Phase 2
- Phase 2 depends on Phase 1 being finished
- Phase 3 can be parallel but cleaner after Phase 1

DOCUMENTS TO READ (in order):
1. AGENT_HANDOFF.md - 10 min (understand project state)
2. 00_START_HERE.md - 5 min (understand guides)
3. CODE_AGENT_COMPREHENSIVE_GUIDE.md - 30 min (all phases overview)
4. QUICK_REFERENCE.md - Print it (keep visible)
5. Specific guides for each phase as you start:
   - Phase 1 → ERROR_CLASSIFICATION.md
   - Phase 2 → ERROR_CLASSIFICATION.md (worker section)
   - Phase 3 → I18N_IMPLEMENTATION.md

VALIDATION GATES (all must pass):
After Phase 1: pytest tests/test_job_error_classification.py -v
After Phase 2: pytest tests/test_worker_retry_logic.py -v
After Phase 3: pytest tests/test_i18n_integration.py -v && npm run i18n:check

TOTAL TIME: 14-18 hours

SUCCESS: All phases complete, all tests passing, WHAT-DONE.md updated

===============================================================================
```

---

## 🎯 VARIATION: FOR FOLLOW-UP AGENTS

### If You're Continuing Previous Work

```
===============================================================================
FOLLOW-UP TASK: Continue Variants Implementation
===============================================================================

WHAT HAPPENED:
- Previous agent completed [Phase X]
- Current status: [describe what's done]
- Pending work: Phase Y onwards

YOUR TASK:
Continue from where [Agent Name] left off

READ THESE FIRST (in order):
1. AGENT_HANDOFF.md - Understand current project state
2. WHAT-DONE.md - See what previous agent completed
3. CODE_AGENT_COMPREHENSIVE_GUIDE.md PHASE [Y] - See what's next

CURRENT STATUS:
- Phases completed: [1, 2, ...]
- Current phase: [Phase X] - [percentage]%
- Blockers: [list any]
- Next immediate task: [specific task]

YOUR DELIVERABLES:
- Complete Phase [X]
- All validation gates passing
- Update WHAT-DONE.md with your work

IMPORTANT:
- Don't repeat previous agent's work (check WHAT-DONE.md)
- Understand previous decisions (AGENT_HANDOFF.md explains reasoning)
- Continue with same patterns and conventions
- Maintain same documentation standards

Commands to run on startup:
cd PX-B && pytest tests/ -v  # See current test status
npm run typecheck            # Check TypeScript
mypy app/                    # Check Python types

Get up to speed in 30 minutes:
1. Read WHAT-DONE.md (what's done)
2. Read AGENT_HANDOFF.md (why it was done)
3. Read relevant section in CODE_AGENT_COMPREHENSIVE_GUIDE.md
4. Ready to code!

===============================================================================
```

---

## ✨ MINIMAL PROMPT (For Quick Handoff)

### If You Just Need a Quick Brief

```
PHASE 1: ERROR CLASSIFICATION

Read these in order:
1. AGENT_HANDOFF.md
2. CODE_AGENT_COMPREHENSIVE_GUIDE.md PHASE 1
3. ERROR_CLASSIFICATION.md

Create:
- PX-B/app/modules/catalog/job_errors.py
- Alembic migration

Test:
- pytest tests/test_job_error_classification.py -v

Update:
- WHAT-DONE.md (after completion, not before)

Check before commit:
- DO_NOT_FORGET.md (115+ item checklist)

Done when:
- All tests pass ✓
- Validation gate passes ✓
- Coverage ≥80% ✓
- WHAT-DONE.md updated ✓
```

---

## 🎓 COMPLETE HANDOFF FLOW FOR NEW AGENT

### Total Time: 2.5 hours to complete Phase 1

**Hour 0-0.5: Orientation (30 min)**
- Read AGENT_HANDOFF.md (10 min)
- Read 00_START_HERE.md (5 min)
- Read CODE_AGENT_QUICKSTART.md (5 min)
- Read IMPLEMENTATION_GUIDE.md (5 min)
- Print QUICK_REFERENCE.md

**Hour 0.5-1: Deep Dive (30 min)**
- Read CODE_AGENT_COMPREHENSIVE_GUIDE.md PHASE 1 (20 min)
- Read ERROR_CLASSIFICATION.md (10 min)

**Hour 1-2: Implementation (60 min)**
- Create job_errors.py (20 min)
- Create alembic migration (10 min)
- Update models.py (10 min)
- Write tests (20 min)

**Hour 2-2.5: Validation (30 min)**
- Run validation gates (10 min)
- Check DO_NOT_FORGET.md (10 min)
- Commit and document (10 min)

---

## 📊 CHECKLIST: Before Handing Off to Agent

**Verify you have:**
- [ ] Described what needs to be done (phase, hours, deliverables)
- [ ] Listed all required documents (in reading order)
- [ ] Provided exact commands to run
- [ ] Explained validation gates (what must pass)
- [ ] Described success criteria (how to know when done)
- [ ] Warned about pitfalls (check DO_NOT_FORGET.md)
- [ ] Linked to CODE_AGENT_COMPREHENSIVE_GUIDE.md (specific phase)
- [ ] Mentioned this is production code (tests required)
- [ ] Explained constraints (no shortcuts, follow patterns)

**Verify agent will:**
- [ ] Read AGENT_HANDOFF.md first (project context)
- [ ] Read CODE_AGENT_COMPREHENSIVE_GUIDE.md for phase (implementation steps)
- [ ] Print QUICK_REFERENCE.md (keep visible)
- [ ] Check DO_NOT_FORGET.md before commits (quality gate)
- [ ] Update WHAT-DONE.md when done (progress tracking)
- [ ] Run all validation gates (prevent failures)

---

## 🚀 YOU'RE READY TO HAND OFF

Use the prompt above to brief code agents.

They'll have:
✅ Clear understanding of what to do  
✅ All documents in reading order  
✅ Code patterns and test patterns  
✅ Validation gates to verify correctness  
✅ Safety checklist to prevent mistakes  
✅ Continuity for next agent (via WHAT-DONE.md and AGENT_HANDOFF.md)  

**This ensures consistent, high-quality implementation across agents.**
