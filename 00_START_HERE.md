# PXPLAN FOLDER - COMPLETE GUIDE INDEX
## Document Navigation for Code Agents

**This folder contains everything a code agent needs to implement production-grade features.**

---

## 📚 DOCUMENT STRUCTURE

### 🎯 START HERE (Read in this order)

#### 1. **CODE_AGENT_QUICKSTART.md** (5 minutes)
**What**: High-level orientation  
**Who**: Every code agent, first thing  
**Contains**:
- What you're building (variants feature, async jobs, SSE)
- Current status (65-70% complete, 3 critical gaps)
- Architecture context (backend + frontend)
- Brief explanation of blockers

**When to read**: FIRST - Get oriented before diving into details

---

#### 2. **IMPLEMENTATION_GUIDE.md** (10 minutes)
**What**: Priority matrix and phase overview  
**Who**: Code agents planning sprint work  
**Contains**:
- 🔴 BLOCKER section (must fix first)
- 🟡 HIGH section (this sprint)
- 🟠 MEDIUM section (before release)
- Task breakdown by file location
- Implementation patterns

**When to read**: SECOND - Understand what needs to be done and why

---

#### 3. **CODE_AGENT_COMPREHENSIVE_GUIDE.md** ⭐ (NEW - READ THIS)
**What**: Complete step-by-step implementation roadmap  
**Who**: Code agents ready to start coding  
**Contains**:
- 5 sequential phases with exact file modifications
- Code patterns and examples
- 🧪 Test patterns for each phase
- Production readiness checklist
- Validation gates (must pass before proceeding)
- File modification table
- Common mistakes to avoid
- Success criteria

**When to read**: THIRD - Use as your implementation bible

---

#### 4. **QUICK_REFERENCE.md** ⭐ (NEW - PRINT THIS)
**What**: One-page checklist you can print and keep visible  
**Who**: Code agents actively coding  
**Contains**:
- Five phases in execution order
- Production readiness gates
- Must-test scenarios
- Common mistakes
- File checklist
- Quick commands

**When to read**: DURING CODING - Refer to constantly

---

### 📖 DETAILED REFERENCE DOCUMENTS

#### 5. **I18N_IMPLEMENTATION.md**
**What**: Internationalization setup guide  
**Who**: Code agents working on i18n (Phase 3)  
**Dependency**: After reading IMPLEMENTATION_GUIDE.md  
**Contains**:
- Backend i18n keys structure
- Frontend i18n config setup
- Translation file organization
- Implementation step-by-step for all 4 languages
- Testing i18n

**When to read**: When implementing Phase 3 (Backend i18n Integration)

---

#### 6. **ERROR_CLASSIFICATION.md**
**What**: Error handling and retry strategy  
**Who**: Code agents working on error classification and retry logic (Phases 1 & 2)  
**Dependency**: After reading IMPLEMENTATION_GUIDE.md  
**Contains**:
- Error classification system design
- JobErrorType enum with descriptions
- Retryable vs non-retryable rules
- Worker retry logic with backoff strategy
- Retry classification rules

**When to read**: When implementing Phase 1 (Error Classification) and Phase 2 (Retry Logic)

---

#### 7. **VARIANTS_FEATURE_STRUCTURE.md**
**What**: Complete codebase map of variants feature  
**Who**: Code agents new to the codebase  
**Dependency**: Optional, but helpful context  
**Contains**:
- Backend structure (database models, schemas, repository, service)
- Frontend structure (components, hooks, state management)
- All variant-related files with line numbers
- Function signatures and responsibilities

**When to read**: If you need to understand existing code structure

---

#### 8. **variant-domain-contract.md**
**What**: Business rules and guarantees for variants  
**Who**: Code agents need to understand business logic  
**Dependency**: Reference during implementation  
**Contains**:
- State machines (structure lifecycle, job lifecycle)
- Variant materialization strategy
- Sync semantics and publication boundaries
- Conflict resolution rules
- Archive vs delete semantics
- Versioning strategy

**When to read**: When implementing variant-related logic (to ensure correctness)

---

#### 9. **technical-architecture-spec.md**
**What**: Technical implementation details  
**Who**: Code agents need to understand system design  
**Dependency**: Reference during implementation  
**Contains**:
- SSE architecture and event schema
- Worker boundaries and communication
- Database contracts
- Snapshot flow
- Retry semantics
- API contracts
- Scalability stages

**When to read**: When implementing SSE, snapshots, or worker features

---

### 📝 PROGRESS TRACKING

#### 10. **WHAT-DONE.md**
**What**: Progress tracking and completed work  
**Who**: Every code agent before and after work  
**Dependency**: Read before starting, update after finishing  
**Contains**:
- Completed slices (what's been done)
- What was implemented in each slice
- Current status of variants feature
- Already-completed work not to redo

**When to read**: BEFORE STARTING - See what's already done  
**Update when**: After completing each phase

---

#### 11. **IMPLEMENTATION_CHECKLIST.md**
**What**: Task-by-task checklist for implementation  
**Who**: Code agents tracking their work  
**Dependency**: Use alongside COMPREHENSIVE_GUIDE.md  
**Contains**:
- 🔴 BLOCKER items with detailed checklists
- 🟡 HIGH priority items
- 🟠 MEDIUM priority items
- Sub-tasks for each item
- Validation gates

**When to read**: Use as detailed task breakdown (more granular than COMPREHENSIVE_GUIDE)

---

### 📋 ADDITIONAL REFERENCE

#### 12. **plan.md**
**What**: Original master plan and architectural vision  
**Who**: Code agents wanting to understand the "why"  
**Dependency**: Reference only  
**Contains**:
- Architectural principles
- Core design decisions
- Phase breakdown (architectural order)
- Variant generation strategy
- Operational limits

**When to read**: When you want context on design decisions

---

#### 13. **work.md**
**What**: Current work in progress notes  
**Who**: Coordinating code agents  
**Dependency**: Optional, context only  
**Contains**:
- Open issues
- Work in progress items
- Known gaps

**When to read**: To understand what's being worked on right now

---

#### 14. **GUIDES_INDEX.md**
**What**: Index of all guidance documents  
**Who**: Navigation reference  
**Contains**:
- List of all guides
- Quick descriptions

**When to read**: If you're lost and need to find a specific document

---

#### 15. **ERROR_CLASSIFICATION.md**
**What**: Deep dive on error handling strategy  
**Who**: Implementing error classification  
**Contains**:
- Error types and classification
- Specific error classes to create
- Retry logic rules

**When to read**: When implementing Phase 1

---

---

## 🗺️ DOCUMENT ROADMAP BY USE CASE

### **I'm a Code Agent Starting Fresh**
1. Read: **CODE_AGENT_QUICKSTART.md** (5 min)
2. Read: **IMPLEMENTATION_GUIDE.md** (10 min)
3. Read: **CODE_AGENT_COMPREHENSIVE_GUIDE.md** (30 min)
4. Print: **QUICK_REFERENCE.md**
5. Start: Phase 1 in COMPREHENSIVE_GUIDE.md

### **I'm Implementing Phase 1 (Error Classification)**
1. Reference: **CODE_AGENT_COMPREHENSIVE_GUIDE.md** Section "PHASE 1"
2. Detailed: **ERROR_CLASSIFICATION.md**
3. Check: **IMPLEMENTATION_CHECKLIST.md** - BLOCKER 2 section
4. Validate: Validation gate in COMPREHENSIVE_GUIDE.md

### **I'm Implementing Phase 2 (Worker Retry)**
1. Reference: **CODE_AGENT_COMPREHENSIVE_GUIDE.md** Section "PHASE 2"
2. Detailed: **ERROR_CLASSIFICATION.md** - Worker Retry Logic section
3. Check: **IMPLEMENTATION_CHECKLIST.md** - BLOCKER 3 section
4. Validate: Validation gate in COMPREHENSIVE_GUIDE.md

### **I'm Implementing Phase 3 (i18n Backend)**
1. Reference: **CODE_AGENT_COMPREHENSIVE_GUIDE.md** Section "PHASE 3"
2. Detailed: **I18N_IMPLEMENTATION.md**
3. Check: **IMPLEMENTATION_CHECKLIST.md** - BLOCKER 1 section
4. Validate: Validation gate in COMPREHENSIVE_GUIDE.md

### **I'm Implementing Phase 4 (Tests)**
1. Reference: **CODE_AGENT_COMPREHENSIVE_GUIDE.md** Section "PHASE 4"
2. Reference: "🧪 CRITICAL TEST PATTERNS" section
3. Check: **IMPLEMENTATION_CHECKLIST.md** - Testing sections
4. Validate: Test commands in COMPREHENSIVE_GUIDE.md

### **I'm Implementing Phase 5 (Dead Code Audit)**
1. Reference: **CODE_AGENT_COMPREHENSIVE_GUIDE.md** Section "PHASE 5"
2. Quick: **QUICK_REFERENCE.md** - PHASE 5 section
3. Validate: Final Gate commands

### **I'm Reviewing Variant Architecture**
1. Read: **variant-domain-contract.md** (business rules)
2. Reference: **technical-architecture-spec.md** (technical design)
3. Reference: **VARIANTS_FEATURE_STRUCTURE.md** (code organization)

### **I Need to Understand What's Already Done**
1. Read: **WHAT-DONE.md** (progress tracking)
2. Reference: **WHAT-DONE.md** - "Completed Slice" sections

### **I'm Stuck or Need to Debug**
1. Check: **QUICK_REFERENCE.md** - "COMMON MISTAKES" section
2. Reference: **variant-domain-contract.md** - Verify business rules
3. Reference: **CODE_AGENT_COMPREHENSIVE_GUIDE.md** - "❌ DON'T DO THIS" section

---

## 📊 DOCUMENT RELATIONSHIP MAP

```
START HERE
    ↓
CODE_AGENT_QUICKSTART.md (orientation)
    ↓
IMPLEMENTATION_GUIDE.md (priorities)
    ↓
CODE_AGENT_COMPREHENSIVE_GUIDE.md ⭐ (detailed roadmap)
    ↓
┌─────────────────────────────────────┐
│ PICK A PHASE (1, 2, 3, 4, or 5)   │
└─────────────────────────────────────┘
    ↓
QUICK_REFERENCE.md (print & keep visible)
    ↓
Detailed guide for your phase:
├─ Phase 1 → ERROR_CLASSIFICATION.md
├─ Phase 2 → ERROR_CLASSIFICATION.md
├─ Phase 3 → I18N_IMPLEMENTATION.md
├─ Phase 4 → IMPLEMENTATION_CHECKLIST.md
└─ Phase 5 → CODE_AGENT_COMPREHENSIVE_GUIDE.md
    ↓
IMPLEMENTATION_CHECKLIST.md (detailed tasks)
    ↓
variant-domain-contract.md (verify business rules)
technical-architecture-spec.md (verify tech design)
VARIANTS_FEATURE_STRUCTURE.md (understand code)
    ↓
UPDATE: WHAT-DONE.md (track progress)
    ↓
PRODUCTION READY ✅
```

---

## 🎓 READING TIME ESTIMATES

| Document | Time | Audience |
|----------|------|----------|
| CODE_AGENT_QUICKSTART.md | 5 min | Everyone |
| IMPLEMENTATION_GUIDE.md | 10 min | Everyone |
| CODE_AGENT_COMPREHENSIVE_GUIDE.md | 30-45 min | Implementers |
| QUICK_REFERENCE.md | 5 min (print) | Implementers |
| I18N_IMPLEMENTATION.md | 15-20 min | Phase 3 |
| ERROR_CLASSIFICATION.md | 15-20 min | Phase 1 & 2 |
| VARIANTS_FEATURE_STRUCTURE.md | 20-30 min | Optional context |
| variant-domain-contract.md | 15 min | Reference |
| technical-architecture-spec.md | 15 min | Reference |
| WHAT-DONE.md | 10 min | Before/after |
| IMPLEMENTATION_CHECKLIST.md | 10-15 min | During work |
| **Total** | **2-3 hours** | Full understanding |

---

## ✅ BEFORE YOU START

**Verify you've read:**
- [ ] CODE_AGENT_QUICKSTART.md
- [ ] IMPLEMENTATION_GUIDE.md
- [ ] CODE_AGENT_COMPREHENSIVE_GUIDE.md
- [ ] Printed QUICK_REFERENCE.md nearby

**Verify you understand:**
- [ ] Current status (65-70% complete)
- [ ] Three blocking issues (error classification, retry, i18n)
- [ ] Five phases in sequential order
- [ ] Validation gates that must pass
- [ ] This is production code - tests are required

**Verify your environment:**
- [ ] `cd PX-B` - Backend ready
- [ ] `cd PX-F/px` - Frontend ready
- [ ] Can run pytest and npm test
- [ ] Can run black, mypy, flake8, eslint
- [ ] Git ready for commits

**If all above checked:** You're ready to start Phase 1.

---

## 🚀 QUICK START COMMAND

```bash
# First time setup
cd /d:/Github/muhsinmuhsy/PX/pxplan

# Read in order
cat CODE_AGENT_QUICKSTART.md | less
cat IMPLEMENTATION_GUIDE.md | less
cat CODE_AGENT_COMPREHENSIVE_GUIDE.md | less

# Print quick reference
code QUICK_REFERENCE.md  # (opens in VS Code)

# Start Phase 1
# Follow CODE_AGENT_COMPREHENSIVE_GUIDE.md "PHASE 1" section
```

---

## 📞 VALIDATION GATES (MUST PASS EACH PHASE)

**Phase 1**: `pytest tests/test_job_error_classification.py -v`  
**Phase 2**: `pytest tests/test_worker_retry_logic.py -v`  
**Phase 3**: `pytest tests/test_i18n_integration.py -v && npm run i18n:check`  
**Phase 4**: `pytest tests/ -v --cov=app --cov-fail-under=80`  
**Phase 5**: `pytest tests/ -v && npm test && npm run lint`  

All must pass before proceeding.

---

## 💡 KEY PRINCIPLE

**This folder contains your complete source of truth for implementation.**

If you have a question while coding:
1. Check QUICK_REFERENCE.md (2-second answer)
2. Check CODE_AGENT_COMPREHENSIVE_GUIDE.md (relevant phase)
3. Check detailed guide for your phase
4. Check variant-domain-contract.md (verify business rules)

**Don't guess. The answer is in these docs.**

---

**Ready? Start with CODE_AGENT_QUICKSTART.md**
