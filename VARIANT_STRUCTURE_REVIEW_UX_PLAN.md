# Variant Structure Review UX Plan

Status: Proposed next UX hardening plan
Created: 2026-05-24
Scope: PX-F product editor Variant Studio, Manage Variants table, maintenance panel, row filters, and copy

This plan extends `VARIANT_TABLE_AND_JOB_UX_PLAN.md` after Slice 33. It focuses on making structural mismatch states understandable for non-technical merchants, especially when the system has current valid variant rows plus previous-option rows that no longer match the current options.

---

## Modern UX for Non-Technical Merchants — PRIMARY FOCUS

**This section establishes the emotional and psychological UX strategy. The architecture is strong; now simplify the merchant experience.**

### The Core Shift: From System Language to Merchant Confidence

Non-technical merchants do NOT think about:
- States, lifecycle, stale rows, operational blocking, classification, rebuild semantics
- Database concepts, occupancy keys, combination matrices, orphaned rows

They think about:
- "Can I continue selling?" ✅
- "Did I lose anything?" ✅ 
- "What should I click?" ✅
- "Is this safe?" ✅

**UX Goal: Merchants understand the page in 3 seconds.**

When opening Manage Variants, they should instantly know:
1. Everything is safe
2. What changed
3. What they should do next

Without reading paragraphs or understanding system concepts.

### Standardized Merchant-First Terminology

Replace all internal/database language with these natural merchant terms. This standardization is critical for premium product feel.

**Primary Merchant Language:**

| Merchant Visible | Use For | Never Use |
| --- | --- | --- |
| **Active variants** | Current, editable, ready to sell | current rows, current variants, live variants |
| **Older variants** | From previous option setup | stale, orphaned, previous-option, needs review, requires action |
| **Archived variants** | Historical, read-only | kept for history, removed, legacy |
| **Create variants** | User action to build new combinations | generate, build, initialize |
| **Recreate variants** | Advanced action to rebuild all | rebuild from scratch, reset |

**Never Expose to Merchants:**
- `rows` or `variant rows` (use "variants")
- `stale_active`, `orphaned`, `structural`, `lifecycle_status`
- `occupancy keys`, `combination keys`, `matrix`
- `mismatch`, `conflict`, `invalid`
- `rebuild semantics`, `previous-option`, `needs review`
- Database metrics, internal counts, aggregate tables

**Terminology Rules:**
1. **UI labels**: Always use standardized terms (Active variants, Older variants, etc.)
2. **Buttons/CTAs**: Use clear verbs (Edit, Create, View, Manage)
3. **Status badges**: Minimal (Active, Older, Archived)
4. **Help text**: Explain human benefit, not technical state (e.g., "Saved safely for history" not "archived due to structural mismatch")
5. **Counts**: Show ONE primary number, ONE secondary only — never dashboard-style metrics

### Psychological UX Principles

**1. Safety First**
- Lead with reassurance, not warnings
- Use ✅ to show safety, not ⚠️ to show problems
- "Saved safely for history" not "Detected orphaned state"

**2. Continuity Over Disruption**
- Normal editing should dominate the screen
- Repair tools should be hidden/collapsed by default
- Merchants should feel: "Everything is okay, continue editing."
- NOT: "You entered a repair workflow."

**3. Visual Clarity Over Metrics**
- Show outcome (243 active), not aggregates (406 total / 163 / 0 / 243)
- Reduce text density 40–50%
- Use cards, not dense tables or metric lists
- Mobile-first simplicity

**4. Progressive Disclosure**
- Show only what the merchant needs to act
- Hide advanced options behind "Advanced tools" / "Maintenance"
- Expand only when clicked, not by default

**5. Emotional Tone**
- Calm, not urgent
- Reassuring, not alarming
- Guided, not overwhelming
- Shopify-like, not enterprise-admin-dashboard-like

### Modern Card-Based Workflow Layout

Replace warning banners and repair panels with clean workflow cards.

**Current (Dense, Warning-Heavy):**
```
⚠️ Product options changed

243 current variant rows are ready to edit.
163 rows from previous options no longer match your current options.
0 new combinations needed.

[Archive previous-option rows and recreate variants]  [Review rows needing review]
```

**Modern (Clean, Confidence-Building):**
```
✅ Your Variants Are Ready

243 active variants ready to edit.

[Continue editing]


ℹ️ Older Variants Kept Safe

163 variants from your previous option setup were saved for history.

[Review if needed]
```

**Key Differences:**
- Lead with active/current (the safe, normal path)
- Use ✅ instead of ⚠️
- Soft informational tone for older variants, not warning tone
- Shorter text blocks
- Clear primary action (Continue/Edit), secondary action (Review)
- No repair language up front

### Visual Hierarchy for Manage Variants Page

```
┌─────────────────────────────────────────┐
│  ✅ Variants Ready                      │  ← Largest, most prominent
│                                          │
│  243 active variants                    │  ← Big number, clear purpose
│  Ready to edit and sell                 │  
│                                          │
│  [Edit variants] [See options]          │  ← Primary actions
└─────────────────────────────────────────┘

┌─────────────────────────────────────────┐
│  ℹ️ 163 Older Variants Saved            │  ← Secondary, informational tone
│                                          │
│  From your previous product options.    │
│  You can review or manage these later.  │
│                                          │
│  [Review] [Collapse]                    │  ← Optional actions
└─────────────────────────────────────────┘

  ⋯ Advanced tools [expand]               ← Hidden by default, collapsed
```

This layout says: "Everything is safe. Continue working. Advanced options are available if needed."

### What to Hide (Keep Out of Merchant View)

Merchants should NEVER see or need to understand:

- `stale_active` state
- `orphaned` or `orphanedCount`
- `occupancy keys` or `combination keys`
- `rebuild semantics` or "rebuild from scratch"
- `lifecycle_status`
- `operational state` or `active_job`
- `incomplete_draft`
- `structural state` classifications
- Database metrics or internal counts
- Repair-heavy language or repair flows by default

These stay in:
- Backend architecture
- Engineering documentation
- Advanced/hidden tooling
- Logs and monitoring
- Code comments

### Mobile-First Simplicity

On mobile (small screens), reduce density even more:

**Desktop:**
```
✅ 243 Active Variants
Ready to edit

[Edit] [Options]

ℹ️ 163 Older Variants
[Review]
```

**Mobile:**
```
✅ Variants Ready

243 active

[Edit]
[More...]
```

Details appear only on expansion, not by default.

### Progressive Disclosure for Advanced Tools

**Default state: Everything simple**
```
Manage Variants

✅ 243 variants ready
163 older variants saved

[Continue editing]
```

**User clicks [More...]:**
```
✅ Variants Ready (243)
[Edit variants]

ℹ️ Older Variants (163)
[Review these variants]

⚙️ Advanced tools
├─ Recreate variants from current options
├─ Archive older variants
└─ Repair failed updates
```

This way:
- 95% of merchants never see repair tools
- Advanced options are there for power users
- The normal path feels simple and safe

### Language Patterns for Different Situations

**Situation: All current, no history**
```
✅ All Set

243 variants are active and ready to sell.
```
(No secondary content. Blank space is okay.)

**Situation: Current + older variants**
```
✅ 243 Active Variants

Ready to edit and sell.

ℹ️ 163 Older Variants

Kept from your previous options. Review anytime if needed.

[Review]
```

**Situation: Missing variants needed**
```
⚠️ Action Needed

Your options changed. Create 12 new variants to match.

[Create missing variants]

You can review previous variants anytime:
[163 older variants]
```

**Situation: Job in progress**
```
⏳ Updating...

Recreating variants from your new options.
This usually takes a few seconds.

(progress bar)
```

**Situation: Job failed**
```
❌ Update Failed

Something went wrong recreating variants. 

[Try again] [Get help]

Previous variants are still saved.
```

### Variant Studio Header — Modern Version

Instead of:
```
Product versions
Editing
243 current versions / 406 total rows / 163 need review
```

Use:
```
Product Variants

243 active · 163 saved for history
```

Smaller, cleaner, metadata-feel.

Or even simpler:
```
Product Variants

Ready to edit
```

(Details in main panel, not header.)

---

## Current Problem (Technical Explanation)

The system can correctly detect when variant rows do not match the current option setup, but the UI explains that state with technical language:

- `Orphaned variants`
- `Removed -163`
- `Structural changes detected`
- `Combination matrix`
- `Show archived`

For a shop owner, this is hard to understand. They need to know:

- Which variant rows are current and safe to edit?
- Which rows are old or archived?
- What changed?
- What is the next safe action?
- Will clicking a repair action delete product history?

Example observed state:

- Current option setup creates `243` valid variants.
- Existing rows show `406`.
- Missing rows show `0`.
- Previous-option rows show `163`.

The correct merchant explanation is:

> All 243 current variant rows are ready. 163 rows from previous options no longer match the current setup and need review.

---

## System Concepts To Preserve

Do not change these backend/domain rules while improving UX:

- Active variant uniqueness remains scoped to `lifecycle_status = active`.
- Standard destructive removals must archive rows, not hard-delete them.
- Storefront reads must remain pinned to `product.published_version`.
- Structural writes must keep active-job and structure-guard protection.
- `variants: null` in lightweight snapshots means rows are not included, not that rows do not exist.
- `orphanedCount` currently means existing occupancy rows whose combination keys do not fit the current option/value structure.
- `include_archived=true` currently means include archived rows with active rows, not show only archived rows.
- Product publish status and variant structure review status are separate. Storefront visibility still depends on published snapshots, not raw draft structure.

---

## Merchant Variant States (Hidden, Internal Architecture)

The backend manages three distinct internal states. These stay hidden from merchants and communicated only through the emotional UX principles above.

| Internal State | Merchant Experience | Merchant Language | Editability |
| --- | --- | --- | --- |
| `current_active` | Active, ready to edit | "Active variant" | Fully editable |
| `stale_active` | Older but preserved | "Older variant" | Limited by policy |
| `archived` | Preserved history | "Archived variant" | Read-only |

**Critical:** These internal names never appear in UI. Merchants see:
- ✅ Active variants (243)
- ℹ️ Older variants saved (163)
- 📦 Archived variants (5)

**Future state (not exposed yet):**
- `incomplete_draft`: When new option combinations need rows, show: "Variants to create" — simple, not technical.

### Editability (Hidden from Merchants, Enforced in Code)

| Internal State | SKU/Price/Stock | Visibility/Media | Bulk Actions | Archive/Restore |
| --- | --- | --- | --- | --- |
| `current_active` | ✓ Editable | ✓ Editable | ✓ Yes | Archive allowed |
| `stale_active` | Limited/conditional | Limited | ✗ No | Archive allowed |
| `archived` | ✗ No | ✗ No | ✗ No | Restore if safe |

**Rule:** Merchants never see "why" a field is disabled. They just see it disabled with a soft helper tooltip.

Bad (technical):
```
✗ Cannot edit: stale_active rows are not editable per policy
```

Good (merchant-friendly):
```
✗ This variant is from an older option setup. 
Review or archive it if needed.
```

Or even simpler, just gray it out silently with light hover help.

---

## Copy Translation Map — Modern Merchant Language

Translate technical/internal language into natural, confidence-building merchant language.

| Technical/Internal | Modern Merchant Language | Example Usage | Tone |
| --- | --- | --- | --- |
| Orphaned variants | Older variants | "163 older variants" | Neutral, historical |
| Removed / Removed -163 | Older variants saved for history | "163 older variants kept safely" | Reassuring |
| Structural changes detected | Your options changed | "You changed your product options" | Factual |
| Missing combinations / Missing rows | Variants to create | "Create 12 new variants" | Action-oriented |
| Generate missing variants | Create missing variants | "Create missing variants" | Clear outcome |
| Rebuild from scratch | Recreate variants from options | "Recreate all variants from current options" | Clear, not scary |
| Show archived | Row view or Include history | Segmented: "Active / Older / Archived / All" | Navigation, not technical |
| Combination matrix | (never expose) | Don't use | — |
| Rows needing review | Older variants or Variants to review | "163 older variants to review" | Optional, not urgent |
| Occupancy / occupancy keys | (never expose) | Use variant, option, combination as needed | — |
| Stale active / Previous-option | Older variant | Badge/card: "Older variant" | Neutral |
| Lifecycle status | (never expose; use context) | "Active variant", "Archived variant" | Context-specific |
| Orphaned count | (never expose) | Show count only: "163 older variants" | Count, no jargon |

**Rules:**
- Keep backend/test names technical where appropriate.
- User-facing copy must ALWAYS use merchant language from this table.
- User-facing copy must ALWAYS be in `admin-copy.ts` i18n pattern.
- Locale dictionaries should inherit safely through existing fallback/spread behavior.
- English source copy must be complete and natural-sounding.

---

## Target UX — Modern, Calm, Confidence-Building

### Manage Variants Top State (Complete Scenario)

**Scenario: Current variants are complete, older variants exist but are not blocking.**

```
┌──────────────────────────────────────┐
│ ✅ Your Variants Are Ready           │
│                                       │
│ 243 active variants ready to edit.    │
│                                       │
│ [Edit Variants] [See Options]         │
└──────────────────────────────────────┘

┌──────────────────────────────────────┐
│ ℹ️ 163 Older Variants Kept Safely    │
│                                       │
│ From your previous option setup.      │
│ You can review or archive these       │
│ anytime if needed.                    │
│                                       │
│ [Review]                              │
└──────────────────────────────────────┘

[⋯ Advanced Tools] (collapsed)
```

**Merchant reads this in 3 seconds and understands:**
- Everything is safe ✅
- 243 are active, ready to sell
- 163 are from old options, kept safely
- [Edit] is the next action
- Advanced tools exist but are hidden

**Old (Dense, Warning-Heavy):**
```
⚠️ Product options changed

243 current variant rows are ready to edit.
163 rows from previous options no longer match your current options.

[Archive previous-option rows and recreate variants] [Review rows needing review]
```

**Why the new version is better:**
- ✅ instead of ⚠️ (confidence, not alarm)
- "Your Variants Are Ready" as headline (positive framing)
- "Active variants" not "current variant rows" (merchant language)
- "Older variants kept safely" not "previous options no longer match" (reassuring, not technical)
- Primary action is [Edit], not [Repair]
- Repair is hidden under Advanced Tools

### Manage Variants — Complete Status Summary

Replace static metrics with clean, scannable card workflow.

**Current table:**
```
| Metric | Count |
| --- | --- |
| Current | 243 |
| Missing | 0 |
| Previous-option | 163 |
| Archived | 2 |
| Total | 408 |
```

**Modern card-based:**
```
✅ 243 Active Variants

Ready to edit and sell.

[Edit Variants]


ℹ️ 163 Older Variants

From previous options. Review anytime.

[Review]


📦 2 Archived

Kept for history.

[View]
```

Much more scannable, modern, Shopify-like.

### Variant Studio Right Panel — Modern Version

**Old:**
```
Removed -163
Structural changes detected
```

**Modern:**
```
ℹ️ Older Variants
163 from previous options

These were created using your 
previous product options and 
are kept safely for reference.
```

Smaller, softer, reassuring, not alarm-inducing.

### Studio Footer CTAs — Simplified

**Situation: Missing variants = 0, Older variants > 0**

Old (confusing):
```
[Generate missing variants (disabled)] [Review rows needing review] [Rebuild]
```

Modern (clear):
```
[Continue Editing] [Review Older Variants]
```

Repair tools are hidden under ⋯ Advanced.

**Situation: Missing variants > 0**

Old:
```
[Create missing variants] [Review rows needing review] [Rebuild]
```

Modern:
```
[Create Missing Variants]

These 12 new combinations from your updated options will be created.

[Review Older Variants] (secondary)
```

Simple, task-focused, no repair language.

---

## Safe Action Hierarchy — Modern Version

Every action should communicate confidence and next step through placement and visual weight.

| Action Type | Example | Visual Placement | Tone | Accessibility |
| --- | --- | --- | --- | --- |
| **Normal editing** | Save variant, change price | Table/inline controls | Default, unstressed | Full interactivity |
| **Safe creation** | Create missing variants | Primary button (top card) | Confident, positive | Large target, clear text |
| **Navigation/optional** | Review older variants, view options | Secondary button | Calm, optional | Available but not forced |
| **Advanced/destructive** | Recreate all, archive, repair | Hidden by default (⋯ Advanced tools) | Muted, collapsed | Requires deliberate access |

**Rules:**
- Editing controls are always visible, never hidden.
- Create/setup actions are primary buttons.
- Repair/destroy actions are NEVER visible on page load.
- Review/navigation actions are secondary/tertiary buttons.
- The page should feel like "continue working" not "fix something."
- Merchants should feel guided, not overwhelmed.

---

## Recommended Next Action Map & Interaction Governance

**This section establishes interaction governance.** The primary CTA should be derived from state, not chosen ad hoc in each component.

**Critical rule:** States map to actions; components do not invent CTAs.

| State Combination | Severity | Merchant Meaning | Primary Action | Secondary Action | Advanced Action |
| --- | --- | --- | --- | --- | --- |
| Current only | Neutral | Everything is ready | None; keep editing | Edit options | Maintenance hidden |
| Missing only | Action required | Setup is incomplete | **Create missing rows** | Edit options | Rebuild hidden/collapsed |
| Previous-option rows only | Non-blocking warning | Current rows are ready, previous rows need review | **Review rows needing review** | Keep editing current rows | Archive previous-option rows and recreate variants |
| Missing + previous-option rows | Action required | Finish current setup first, then review previous rows | **Create missing rows** | Review rows needing review | Rebuild collapsed |
| Archived only included | Informational | User chose to view history | Continue editing current rows or switch view | Restore selected archived row when safe | None |
| Active job running | Processing | Background update in progress | Wait/show progress | None | All structure actions disabled |
| Failed job | Error | Last update failed | **Retry or review error** | Keep safe current edits if allowed | Repair collapsed |

**Governance Notes:**
- This table **is the source of truth** for CTA selection.
- Components query state → look up primary action → render it.
- Tests verify that all paths follow this map.
- Product decisions about new states update this table first, then components follow.
- No component should implement its own action prioritization logic.

**Critical Decision Points:**
- `Create missing rows` always outranks repair when missing combinations exist.
- `Review rows needing review` becomes primary only when there are no missing rows but previous-option rows exist.
- Repair/rebuild actions are **tertiary or hidden** on the everyday table.
- Publish and row-review CTAs remain separate concerns.

---

## Row View / Filtering UX

Replace the single `Show archived` checkbox with a segmented row view.

Recommended options:

- `Current`
- `Needs review` or `Previous-option rows`
- `Archived`
- `All`

Optional counts:

- `Current 243`
- `Needs review 163`
- `Archived 2`
- `All 408`

Behavior:

- `Current` shows only rows that match current options and are active.
- `Needs review` shows rows outside the current option setup.
- `Archived` shows archived rows.
- `All` shows grouped sections.
- Default view is always `Current` when rows needing review or archived rows exist.
- `Needs review` should not silently include archived rows unless the UI explicitly says so.
- If usability testing shows `Needs review` is too vague, use `Previous-option rows` as the tab label and keep `Needs review` as the status badge/helper language.

If full classification cannot be delivered immediately, phase it:

1. Rename `Show archived` to `Include archived rows`.
2. Add helper text: `Archived rows appear together with active rows.`
3. Later replace with segmented row views once row classification is available.

---

## Table Presentation

Rows should show clear status badges.

Current row badge:
- `Current`
- Green/neutral tone.

Needs review badge:
- `Needs review`
- Amber tone.
- Helper: `No longer matches current options.`

Archived row badge:
- `Archived`
- Muted amber/gray tone.
- Helper: `Kept for history.`

Avoid mixing archived rows into the table without explaining why they are visible.

If `All` is selected, group rows:

```text
Current variants
...

Rows needing review
...

Archived variants
...
```

---

## Technical Fit

Current frontend has:

- `combinationState.orphanedCount`
- `combinationState.missingCount`
- `combinationState.possibleCount`
- `combinationState.currentCount`
- `VariantIdentityCell`
- `VariantMaintenancePanel`
- `VariantStructureStudio`
- `includeArchived` loading via `useVariantPagination`
- row lifecycle via `variant.lifecycle_status`

Potential gap:
- The table currently may not know whether each active row is current or orphaned.
- `orphanedCount` exists at aggregate level through occupancy, but per-row classification may need a frontend helper or backend projection.

Preferred incremental frontend helper:

- Build a current valid value-id lookup from `options`.
- For each loaded variant row, classify:
  - `current` if its selection IDs form one complete valid option/value combination.
  - `needs_review` if not.
  - `archived` if `lifecycle_status === archived`.

This classification should reuse canonical variant key helpers from `variant-domain.ts` / `variant-matrix.ts` and avoid ad hoc key parsing.

Possible later backend improvement:

- Add row classification to variant list response if frontend-only classification becomes expensive or inconsistent for paginated data.

---

## Formalized Row Capability Selectors

**Critical Architectural Principle:**

> "Do not let each component rediscover this logic independently."

To prevent inconsistent bulk actions, unsafe inline edits, broken restore flows, and contradictory permissions, **centralize editability and capability rules in reusable selectors**.

### Centralized Selector Functions

Create reusable helpers in `variant-domain.ts` or new `variant-capabilities.ts`:

```typescript
// Editability selectors
canEditRowFields(row: ProductVariantRead, state: RowClassification): boolean
canBulkEditRows(rows: ProductVariantRead[], state: RowClassification[]): boolean
canArchiveRow(row: ProductVariantRead): boolean
canRestoreRow(row: ProductVariantRead): boolean
canAssignMediaToRow(row: ProductVariantRead): boolean

// Feature availability selectors
canDeleteRow(row: ProductVariantRead): boolean
canBindAttributes(row: ProductVariantRead): boolean

// Decision helpers
getPrimaryActionForState(combinationState: VariantCombinationState, operationalState: OperationalState): CTAType
getEditabilityReason(row: ProductVariantRead, action: EditAction): EditabilityReason
```

### Implementation Rules

| Row Status | SKU/Price/Stock | Visibility/Media | Bulk Actions | Restore/Archive | Reason |
| --- | --- | --- | --- | --- | --- |
| `current_active` | ✓ Yes | ✓ Yes | ✓ Yes (safe) | Archive allowed | Matches current setup |
| `stale_active` | Limited/conditional | Limited | ✗ No by default | Archive allowed | No longer matches options |
| `archived` | ✗ No | ✗ No | ✗ No | Restore when safe | Preserved for history |

**Key Invariants:**
- A single `canEditRow()` call replaces scattered condition checks.
- Components call `canBulkEditRows()` to validate before bulk operations.
- Tests verify that all components use the same selectors.
- Future permission layers extend selectors, not components.

---

## State Severity Model

Define explicit severity semantics so that UI styling, banner prominence, and CTA weight derive from severity, not ad hoc design choices.

| State | Severity Level | Meaning | UI Treatment | CTA Weight | Badge Tone |
| --- | --- | --- | --- | --- | --- |
| `current` | Neutral | All systems go; ready to edit | Quiet, normal typography | N/A | None or soft green |
| `missing_rows` | **Action required** | Current setup incomplete | Warning banner | **Primary** | Amber |
| `stale_active` (previous-option) | **Non-blocking warning** | Rows exist but no longer match | Informational card | Secondary | Amber |
| `archived` | Informational | Kept for history | Muted styling | N/A | Gray |
| `failed_job` | **Error** | Last operation failed | Error banner/panel | Tertiary (retry/review) | Red |
| `active_job` | Processing | Background operation in progress | Progress indicator | N/A (blocked) | Blue/purple |

**Severity Determines:**
- Color intensity and tone
- Banner vs card vs badge presentation
- Toast priority and dismissibility
- CTA prominence in footer/header
- Whether action blocks editing

**Non-blocking Warning Philosophy:**

Most critical: `stale_active` (previous-option rows) are **non-blocking**.
- Current rows remain editable.
- Repair is not forced.
- User can safely continue working.
- Review is encouraged, not required.

This keeps merchant confidence high and prevents "system is broken" messaging.

---

## Naming Convention Refinement

Eliminate ambiguity by using context-sensitive labels.

| Context | Recommended Label | Reason | Examples |
| --- | --- | --- | --- |
| **Tab/filter selector** | `Previous-option rows` | Descriptive, navigation-focused | "Click here to see rows from previous options" |
| **Inline status badge** | `Needs review` | Action-oriented, concise | Appears on row; tells user what to do |
| **Helper/explanatory text** | `Created from previous product options` | Full clarity for tooltips | "These rows were created using options that have since changed. They are kept safely for review." |
| **Tab count indicator** | `Previous-option rows 163` | Total count for this view | Segmented filter shows active/previous/archived counts |

**Rationale:**
- Navigation labels should describe **what** the user will see.
- Inline badges should suggest **what to do**.
- Help text should **explain why**.
- This pattern scales to future states like `incomplete_draft`.

Avoid throughout:
- `Orphaned` (technical jargon)
- `Removed` (implies deletion, which isn't accurate)
- `Structural mismatch` (developer language)
- `Invalid combination` (scary to merchants)

---

## Future Row Source Metadata

**Not for merchants. For internal tooling.**

If row history, debugging, or migration becomes important, add optional metadata:

```python
# In ProductVariant model (future)
class ProductVariant(SQLModel, table=True):
    # Existing
    id: UUID
    product_id: UUID
    combination_key: str
    lifecycle_status: str
    
    # Future metadata (optional, non-merchant-visible)
    origin_structure_version: int | None = None  # Version when row was created
    created_from_option_snapshot: dict | None = None  # Snapshot of options at creation
    last_rebuild_job_id: UUID | None = None  # Last job that affected this row
```

**Enables:**
- Debug "why was this row created this way?" for support.
- Audit trails for compliance.
- Analytics on row lifecycle patterns.
- Diff viewers: "what changed since rebuild?"
- Migration tooling: smart revert logic.

**Does not change:**
- Merchant UI or copy.
- Row editability or visibility rules.
- API contracts or v1 responses.

**Implementation approach:**
- Add as optional columns in phase 6+.
- Populate during `execute_catalog_variant_job()`.
- Expose only via admin debug endpoints, not public APIs.
- Document in backend architecture, not user guides.

---

## Implementation Phases

### Phase 1: Language Alignment

Goal:
- Make current states understandable without changing backend contracts.

Tasks:
- Update English source copy in `admin-copy.ts`.
- Replace primary UI language:
  - `Orphaned variants` -> `Rows needing review`
  - `Removed` -> `Needs review` or `Would be archived`
  - `Show archived` -> `Include archived rows`
- Add helper copy for why previous-option rows appear.
- Update Studio, Manage Variants, and Maintenance copy consistently.
- Add/adjust i18n copy tests.

Acceptance:
- A non-technical user can tell that current rows are safe and previous-option rows need review.

### Phase 2: Clear Status Summary Cards

Goal:
- Replace developer-style metrics with merchant status summaries.

Tasks:
- Add a top summary component for Manage Variants:
  - current rows ready,
  - missing rows needed,
  - rows needing review,
  - archived rows included when relevant.
- Drive the summary primary CTA from the Recommended Next Action Map.
- Update Studio right panel to use `Needs review` language.
- Hide advanced repair metrics until maintenance is opened.

Acceptance:
- The first visible message says what happened and what to do next.

### Phase 3: Row View Filter

Goal:
- Make row visibility obvious.

Tasks:
- Replace `Show archived` checkbox with segmented row view:
  - `Current`
  - `Needs review`
  - `Archived`
  - `All`
- If per-row orphan classification is not yet ready, start with:
  - `Active`
  - `Archived`
  - `All`
  and add `Needs review` in Phase 5.
- Persist row view only locally unless product requirements need URL state.

Acceptance:
- Users understand why active and archived rows appear together.

### Phase 4: Repair Flow Refinement

Goal:
- Make repair actions safe and understandable.

Tasks:
- Rename rebuild action to explain outcome:
  - `Archive previous-option rows and recreate variants`
- Add confirmation copy:
  - how many current rows stay,
  - how many previous-option rows will be archived,
  - whether media bindings or attributes are affected.
- If `missingCount === 0`, do not show a disabled generate button as the primary action.
- Primary action for previous-option-row-only state should be `Review rows needing review`.
- Add a `Why am I seeing this?` helper to the review summary and Studio.
- Apply the safe action hierarchy so destructive repair actions are danger-tertiary and collapsed by default.

Acceptance:
- Users understand that history is preserved and what the repair action changes.

### Phase 5: Per-Row Classification

Goal:
- Show status on every variant row.

Tasks:
- Add a reusable classification helper.
- Add reusable editability helpers derived from row status.
- Feed row status into `VariantIdentityCell` or `VariantTableRow`.
- Add badges:
  - `Current`
  - `Needs review`
  - `Archived`
- Group rows when `All` is selected.
- Keep selection/bulk updates limited to editable/current active rows by default.

Acceptance:
- Previous-option rows are visible only in the expected view/group and clearly labeled.

Note:
- This phase is intentionally later because it is the most technically invasive. Copy, summary hierarchy, row view, and repair wording deliver the largest UX improvement first.

---

## Published vs Draft Awareness

PX already separates admin draft structure from storefront published structure. This plan should not make that boundary louder than necessary, but the UI should avoid implying that every admin edit is already live.

Future-friendly helper copy:

```text
Storefront uses the last published version. Recent option and variant changes stay in draft until you publish them.
```

Use this only near publish-related UI or when structure changes are pending publication. Do not mix product publish state with row review status.

---

## Testing Strategy

Frontend unit/component tests:

- `product-variants-section.test.tsx`
  - summary for current rows + needs-review rows,
  - row view segmented control,
  - archived rows are not implied by `Show archived` wording,
  - default view is `Current` when review/archive rows exist,
  - maintenance remains secondary.

- `variant-structure-studio.test.tsx`
  - previous-option-row-only state shows `Review rows needing review`, not disabled generate,
  - Studio right panel says `Needs review`, not `Removed`,
  - missing-row state still shows create-missing primary action.

- `variant-matrix.test.ts`
  - per-row classification helper if added to catalog lib.

- `ui-copy.test.ts`
  - required copy exists across locale fallback structure.

Backend tests:

- Not required for Phase 1/2 copy-only frontend changes.
- Required only if adding backend row classification or changing variant list API.

Validation:

```powershell
cd PX-F
npm test
npm run lint
npm run typecheck
npm run i18n:check
npm run build
```

If backend API/schema changes are introduced:

```powershell
cd PX-B
.\w.venv\Scripts\python.exe -m pytest tests -q
.\w.venv\Scripts\ruff.exe check .
.\w.venv\Scripts\mypy.exe app
```

---

## Interaction Governance Framework

**This is the critical architectural achievement of this document.**

Before this section, we had copy guidance and design patterns.

**Now** we have **interaction governance:** the system behaves according to deterministic rules, not individual component intuition.

### The Governance Principle

When you read this statement:

> "Primary CTAs follow the Recommended Next Action Map."

You have crossed from UX documentation into **system architecture**.

This means:
- State is known.
- State determines action.
- Action is rendered.
- **The mapping is testable.**
- **The mapping is centralized.**
- **Future contributors cannot deviate.**

### What This Prevents

Without governance, systems accumulate:

- Different components showing different CTAs for the same state.
- Dangerous repair actions becoming visually prominent by accident.
- Non-blocking warnings treated as critical errors.
- Confusing permission rules spread across 12 files.
- Future PRs adding features that violate the action hierarchy.

With governance:

- State → Lookup → Render. Done.
- Tests catch deviations immediately.
- The product team updates the map; components follow.
- New states extend the map; they don't create new rules.

### Implementation Checklist

Before any component renders a CTA:

- [ ] Component queries operational state and combination state.
- [ ] Component looks up primary action in the Recommended Next Action Map.
- [ ] Component uses centralized selectors for editability (from row capability section).
- [ ] Component checks severity level to determine styling.
- [ ] Component uses centralized naming (from Naming Convention section).
- [ ] Tests verify that the CTA matches the map for all state combinations.
- [ ] No hard-coded UI logic for "should this row be editable?" exists in the component.

### Benefits at Each Layer

| Layer | Benefit |
| --- | --- |
| **Merchant** | Consistent, predictable UI. Knows what to do next. Trusts the system. |
| **Product team** | Single source of truth. Clear when policy changes. Easy to review PRs. |
| **Engineering** | No ambiguity. Centralized logic. Easy to test. Easy to debug. |
| **Future maintainers** | No detective work. Rules are explicit. Adding features is adding to the map, not guessing in UI code. |

---

## Operational vs Structural State Separation

**Critical formalization to prevent architectural drift.**

The system manages three independent state dimensions. Conflating them will cause permission logic, blocking behavior, and action logic to scatter.

### State Categories

| Category | Examples | Drives | Concerns |
| --- | --- | --- | --- |
| **Structural state** | `current` / `stale_active` / `incomplete_draft` | Row classification, editability, visibility filters | Do rows match current setup? |
| **Operational state** | `idle` / `active_job` / `failed_job` / `processing` | UI blocking, polling, progress display | Is a background job running? |
| **Lifecycle state** | `active` / `archived` | Row retention, historical queries, restore eligibility | Is the row active or preserved? |
| **Publish state** | `draft` / `published` | Storefront visibility, admin/merchant split | What version is live on storefront? |

### Why Separation Matters

**Without clear separation:**
- Structural decisions (can I edit this row?) get tangled with operational decisions (is a job running?).
- Loading states become mingled with architecture states.
- Concurrent operations (retries, optimistic updates, background rebuilds) become ambiguous.
- Future permission layers cannot extend cleanly.

**With separation:**
- Each dimension has a single responsibility.
- Rules compose cleanly: "this row is structurally current AND no job is running → can edit."
- Async operations stay orthogonal to structural definitions.
- New states (retry, rollback, conflict) fit naturally into operational dimension.

### Structural State Rules

Structural state determines:
- Which rows display in which view/filter.
- Whether a row is editable (safe or limited).
- What row-level actions are available.

Rules are **independent of operational state** (except blocking under active jobs):

```typescript
// Structural state does NOT change when a job is running.
// Structural state does NOT change when a publish is pending.
// Structural state changes ONLY when options/variants are modified.

getStructuralState(row: ProductVariantRead, currentOptions: ProductOption[]): 'current' | 'stale_active' | 'archived' | 'incomplete_draft'
```

### Operational State Rules

Operational state determines:
- Whether the UI is locked for modifications.
- What progress/blocking UI to display.
- When to poll or subscribe for updates.

Rules are **independent of structural state**:

```typescript
// Operational state affects ALL rows equally (entire product is locked during job).
// Operational state is transient (changes as job progresses).
// Operational state drives UI/UX presentation, not data model.

getOperationalState(job: CatalogVariantJobRead | null, activeJobStatuses: string[]): 'idle' | 'queued' | 'processing' | 'completed' | 'failed'
```

### Compose for Complete Permissions

Editability decisions compose cleanly:

```typescript
canEditRow(row: ProductVariantRead, currentOptions: ProductOption[], activeJob: CatalogVariantJobRead | null): boolean {
  const structuralState = getStructuralState(row, currentOptions);
  const operationalState = getOperationalState(activeJob, ACTIVE_STATUSES);
  
  // Operational blocking takes precedence
  if (operationalState !== 'idle' && operationalState !== 'completed') {
    return false; // Active job blocks all edits
  }
  
  // Structural rules apply
  if (structuralState === 'current_active') {
    return true; // Current rows always editable (when operational is idle)
  }
  if (structuralState === 'stale_active') {
    return false; // Stale rows limited (product policy)
  }
  if (structuralState === 'archived') {
    return false; // Archived rows never editable
  }
  
  return false;
}
```

---

## Blocking Behavior Rules

**Explicit specification to prevent hidden UI states.**

Severity indicates priority and visual weight. Blocking behavior indicates whether the user can proceed.

### Blocking Behavior Matrix

| State | Blocks Editing? | Blocks Publishing? | Blocks CTA Clicks? | Visual Indicator | UI Affordance |
| --- | --- | --- | --- | --- | --- |
| `current` | No | No | No | None | Full controls enabled |
| `stale_active` | No | No | No | Badge only | Full controls enabled |
| `incomplete_draft` | No | No | No | Warning card | Primary action enabled |
| `active_job` | **Yes, partial** | **Yes** | **Partial** | Progress bar | Controls disabled, poll active |
| `failed_job` | No | **Yes** | No (enable retry) | Error banner | Retry CTA enabled |
| `hard_conflict` (future) | **Yes, full** | **Yes** | **Yes** | Modal/overlay | Forced user action |

### Blocking Rules

**Rule: Active job blocks all variant edits on that product.**
- Rationale: Cannot safely edit while rebuild is running.
- Implementation: Check `activeVariantJob?.status` in `isActiveVariantJobStatus()`.
- Scope: Product-wide; all variants locked.
- Release: Job completion OR cancellation.

**Rule: Incomplete draft does NOT block editing.**
- Rationale: Can still edit existing current rows while missing rows exist.
- Implementation: `incomplete_draft` state → warning, not blocker.
- Scope: Row-level; missing rows cannot be edited (don't exist yet).
- Release: Always available; managed via "Create missing rows" CTA.

**Rule: Stale active does NOT block editing current rows.**
- Rationale: Current rows are safe; stale rows are separate review flow.
- Implementation: `canEditRow()` checks structural state separately.
- Scope: Row-level; only stale rows have limited editing.
- Release: Context-dependent (product policy).

**Rule: Publish cannot proceed if active job is running.**
- Rationale: Publishing must capture stable variant structure.
- Implementation: Disable publish UI when `activeVariantJob?.status` is active.
- Scope: Product-wide.
- Release: Job completion.

### UI Affordance Rules

| Blocking Status | UI Treatment | Example |
| --- | --- | --- |
| No blocking | Normal controls | Editable input, clickable button |
| Non-blocking warning | Disabled appearance, informational badge | Muted styling, amber badge, helper text |
| Partial blocking (job active) | Disabled controls + progress | Grayed inputs, progress bar, "Working..." indicator |
| Full blocking (hard conflict) | Modal or overlay | Cannot dismiss, forced resolution |

### UI Semantics: Disabled vs Blocked vs Hidden

Clarify control state meaning for consistent UX and accessibility.

| State | Meaning | User Perception | Implementation | Accessibility |
| --- | --- | --- | --- | --- |
| **Enabled** | Available, user can interact | "I can do this" | Cursor: pointer; opacity: 100% | Focusable, clickable |
| **Disabled** | Unavailable by policy | "This never applies to me" | Opacity: 50%; cursor: not-allowed | aria-disabled=true; explain why in tooltip |
| **Blocked** | Temporarily unavailable (transient) | "This is locked right now" | Opacity: 60%; cursor: wait; loading indicator | aria-disabled=true; polling active; auto-enable when unblocked |
| **Hidden** | Not relevant in this context | "This doesn't apply to my workflow" | display: none | Not in tab order |

**Application Examples:**

| Component | State | Semantics | Example |
| --- | --- | --- | --- |
| SKU input on stale row | Disabled | "Stale rows cannot be edited per policy" | Grayed, explain in helper text |
| Save button during job | Blocked | "Cannot save; job is running" | Grayed with spinner, will enable when job finishes |
| Rebuild action (safe flow) | Hidden | "Not relevant; no previous-option rows exist" | Not rendered; not in tab order |
| Publish button with active job | Blocked | "Cannot publish while rebuild running" | Grayed with progress, will enable when job finishes |

**Important Distinction:**

- **Disabled = Policy**: "This row type doesn't support this action. Ever."
- **Blocked = Transient**: "This action is locked right now, but will unlock automatically when conditions change."
- **Hidden = Irrelevant**: "This action doesn't apply to your current situation."

**Accessibility Requirements:**

- All disabled/blocked controls must have `aria-label` or tooltip explaining why.
- Blocked controls should indicate how to unblock: "Waiting for background job to complete" with estimated time if available.
- Hidden controls should not appear in DOM; do not use `visibility: hidden`.

---

## Source-of-Truth Hierarchy

**Prevents duplication and ensures consistent ownership.**

When multiple engineers touch variant logic, explicit ownership prevents accidental duplication and architectural drift.

| Concern | Source of Truth | Location | Ownership | Updates When |
| --- | --- | --- | --- | --- |
| Row classification | `getStructuralState()` | `variant-domain.ts` | Catalog lib team | Options/variants modified |
| Editability rules | `canEditRow()`, `canBulkEditRows()` | `variant-capabilities.ts` (new) | Product engineering | Policy changes |
| CTA prioritization | Recommended Next Action Map | This document (table) | Product team | New state dimensions added |
| Severity semantics | Severity model (table) | This document + severity constants | Design system | Brand/UX guidelines evolve |
| Blocking behavior | Blocking behavior rules (table) | This document + implementation | Product engineering | Async patterns grow |
| Operational state | `getOperationalState()` | `variant-domain.ts` | Catalog lib team | Job execution changes |
| UI copy | `admin-copy.ts` | `lib/catalog/admin-copy.ts` | Localization team | Feature changes, translations |
| Visual styling | Severity + affordance rules | `variant-styles.css` / tailwind | Design system | Severity model updates |

### Enforcement

**Before implementing a new feature:**

1. Check: Is my concern already owned somewhere in this hierarchy?
2. If yes: Use the existing source-of-truth (call the selector, query the map, apply the styling).
3. If no: Propose new concern ownership in this table and document it.
4. Never: Duplicate logic, create local selectors, hard-code styling derived from severity.

**During code review:**

- Check every permission/blocking/CTA decision against this hierarchy.
- If the code doesn't call a centralized source-of-truth, request refactor.
- If the hierarchy is missing an ownership entry, add it.

### Benefits

| Layer | Benefit |
| --- | --- |
| **Merchant** | Consistent behavior, predictable UI across product. |
| **Product** | Single source of truth for each concern; easy to review changes. |
| **Engineering** | Clear ownership; no confusion about where logic lives; easy to refactor. |
| **Localization** | Copy changes flow through one file; consistent translations. |
| **Design** | Styling logic flows from documented severity model; no ad hoc colors. |

---

## Future Operational State Scope Considerations

**Not a current problem. For architectural awareness.**

Currently:

```
active_job blocks all variant edits on the product
```

This is correct because variant rebuild is product-wide.

### Future Scenarios Where Scope May Change

As the system grows, operational state scope might need refinement:

| Scenario | Current Model | Future Possibility | Complexity |
| --- | --- | --- | --- |
| Background media rebinding | Product-wide lock | Row-level lock | Medium |
| Partial rebuild (subset of options) | Product-wide lock | Scoped rebuild lock | High |
| Batch imports | Product-wide lock | Batch-scoped lock | High |
| Async attribute updates | Product-wide lock | Row-level async | Medium |
| Concurrent socket updates | Product-wide lock | Conflict detection | Very high |

### When to Revisit

Revisit operational state scope if:
- Users request non-blocking background operations (media, attributes).
- Product requires partial rebuilds (subset of variants).
- Multi-user concurrent editing becomes a requirement.
- Performance analysis shows blocking all rows hurts UX.

### Safe Default

Keep product-wide blocking for now:
- Rebuild is rare enough that blocking entire product is acceptable.
- Simplifies consistency and conflict prevention.
- Move to scoped locking only if performance/UX data justifies complexity.

---

## Finite State Machine Architecture (Future Direction)

**Not for immediate implementation. For architectural awareness.**

The current specification naturally evolves toward FSM semantics:

```
State → Severity → Blocking Behavior → Permissions → Recommended Action → UI Rendering
```

This is already mechanically an FSM.

### Eventual Formalization

As the system grows (async jobs, optimistic UI, retry flows, rollback), consider formalizing:

```typescript
// Pseudocode: Future architecture direction
type VariantReviewFSM = {
  initial: 'idle';
  states: {
    idle: {
      on: { START_GENERATION: 'generating', START_REBUILD: 'rebuilding' };
      actions: { renderNormalUI, enableEditing };
    };
    generating: {
      on: { SUCCESS: 'success', FAILURE: 'failed', CANCEL: 'idle' };
      actions: { renderProgress, disableEditing };
    };
    rebuilding: {
      on: { SUCCESS: 'success', FAILURE: 'failed', CANCEL: 'idle' };
      actions: { renderProgress, disableEditing };
    };
    success: {
      on: { ACKNOWLEDGE: 'idle' };
      actions: { renderSuccess, refreshData };
    };
    failed: {
      on: { RETRY: 'generating' | 'rebuilding', DISMISS: 'idle' };
      actions: { renderError, enableRetry };
    };
  };
};
```

**Not required now**, but the architecture is naturally moving here. Each new state dimension would extend this model cleanly.

**Benefits of eventual FSM formalization:**
- Impossible states become impossible to represent.
- Transitions are explicit and testable.
- Error handling is centralized.
- Concurrent operations compose predictably.
- Future contributors add states to the machine, not scattered logic.

---

## Design Acceptance Criteria

- The UI never requires users to understand `orphaned`, `matrix`, `snapshot`, or `combination_key`.
- The first message in Manage Variants explains:
  - current rows,
  - rows needing review,
  - next safe action.
- Default table view is `Current` when rows needing review or archived rows exist.
- `Needs review` and `Archived` remain separate row states internally.
- Studio and Manage Variants use the same language for the same state.
- A checked control never implies “show only archived” when it actually means “include archived.”
- Destructive/advanced actions are clearly separated from everyday editing.
- Empty, loading, current, missing, needs-review, archived, and failed states are visually distinct.
- Current rows stay editable when previous-option rows need review unless a real backend safety rule blocks editing.
- Row editability is centralized and testable; components do not invent their own rules.
- **Primary CTAs follow the Recommended Next Action Map** — state → action mapping is deterministic and testable.
- Severity levels determine visual treatment: color, banner type, CTA weight, whether actions are blocked.
- Non-blocking warnings (`stale_active` / previous-option rows) do not prevent current row editing or suggest system failure.
- Centralized selectors (`canEditRow()`, `canBulkEditRows()`, etc.) are the single source of truth for permissions.
- Tests verify that all components use centralized selectors; ad hoc permission logic is forbidden.
- Naming is consistent by context: tabs use "Previous-option rows", badges use "Needs review", help text explains origin.
- Row source metadata is prepared (backend) for future use in debug/audit/analytics but does not impact merchant UI.
- Interaction governance is enforced: no component implements its own CTA logic; all queries go through the action map.
- **Operational and structural states are cleanly separated:** structural state determines row classification; operational state determines UI blocking.
- Blocking behavior is explicit and testable: rules specify which states block editing, publishing, or CTA clicks.
- Blocking rules are enforced: active job blocks all variant edits on the product; incomplete draft does not block current row editing.
- UI affordances follow blocking rules: disabled controls for active jobs, informational styling for non-blocking warnings, error styling for failures.
- Architectural evolution is prepared: current state composition supports future FSM formalization if concurrency/retry patterns grow.

---

## Modern Merchant-Centric UX — Refined Acceptance Criteria

**Strategic Shift:** From "architecture quality" to "emotional simplicity for non-technical merchants."

### Highest Priority: Emotional & Psychological UX

✅ **Merchants understand the entire page in 3 seconds** without reading explanations or grasping system concepts.

✅ **First visible content communicates two things only:**
1. "Everything is safe ✅"
2. "What to do next"

✅ **Language is always merchant-natural, never system-technical:**
- ✅ Use: "243 active variants", "Older variants kept safely", "Create missing variants"
- ❌ Never expose: `orphaned`, `stale_active`, `lifecycle`, `occupancy`, `rebuild`, `structural`, `matrix`

✅ **Visual tone is calm and confidence-building**, not warning-heavy or alarming:
- ✅ Checkmark icons (✅), informational cards (ℹ️), soft colors
- ❌ Warning banners (⚠️), error tones, repair-feeling sections (unless actual failure)

✅ **Normal editing dominates the screen** — merchants feel: "Everything is okay. Keep working."
- Primary visible actions: [Edit], [Continue], [Create missing variants]
- Secondary actions: [Review], [Details], [See options]
- Advanced/destructive actions: Hidden under [⋯ Advanced Tools]

✅ **Repair/destructive tools are hidden by default** — never expose on page load.

✅ **Text density is reduced 40–50%** from current version:
- Replace dense warning paragraphs with short info cards
- Replace metric tables with scannable card layout
- Use progressive disclosure for complex information

✅ **Mobile-first simplicity** — all complexity hidden until explicitly expanded.

✅ **Modern card-based layout** (Shopify-like, not admin-dashboard):
- Safe/ready state in large, prominent card
- Secondary info in smaller information card
- Advanced options completely hidden/collapsed

✅ **Terminology is consistent in merchant UI:**
- ✅ Always: "243 active variants", "Older variants", "Create variants"
- ❌ Never: "243 current variant rows", "rows needing review", "generate"

✅ **Empty, error, and processing states are visually distinct but not alarming** — use progress indicators, not error language.

### Engineering Architecture (Hidden But Fully Enforced)

✅ The UI never exposes to merchants: `orphaned`, `matrix`, `snapshot`, `combination_key`, `lifecycle_status`, `operational state`, `stale_active`, `occupancy`.

✅ Default table view is `Active` (or `Current`) when older/archived variants exist.

✅ `Older variants` and `Archived` remain separate row states internally (for filtering, bulk actions, analytics).

✅ Row editability is centralized in `canEditRow()`, `canBulkEditRows()` selectors — **no ad hoc permission logic in components**.

✅ **Primary CTAs follow the Recommended Next Action Map** — state automatically determines the primary action.

✅ Severity levels drive visual treatment (color, prominence, CTA weight, blocking behavior).

✅ Older variants (`stale_active`) are **always non-blocking** — merchants can edit current variants even with older variants present.

✅ Naming is consistent by context:
- Tabs/filters: "Active variants", "Older variants", "Archived variants"
- Inline badges/labels: "Active", "Older", "Archived"
- Tooltips/help: "Created from previous product options"

✅ **Visible UX is simple; hidden architecture is strong** — engineering excellence should be transparent through effortless merchant experience.

---

## Open Questions

1. Should `Needs review` include archived rows that no longer match current options, or should archived rows always stay only under `Archived`?
2. Should the default table view be `Current` when previous-option rows exist?
3. Should row-view state persist in URL query params for refresh/shareability?
4. Do we need a backend `row_status` projection for performance once products have thousands of variants?
5. Should stale active rows allow direct SKU/price/stock edits, or should they be review/archive-first only?

Recommended default answers:

- Default view must be `Current`.
- `All` should group current, needs-review, and archived rows.
- Archived rows should remain visually separate from active needs-review rows.
- Start with frontend classification; move to backend projection only if needed.
- Centralize editability and recommended-action decisions in helpers/selectors before spreading logic into UI components.
