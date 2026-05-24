# Merchant Variant UX Plan (Safe, Calm, & Simple)

Status: Proposed next UX hardening plan
Created: 2026-05-24
Scope: PX-F product editor Variant Studio, Manage Variants table, maintenance panel, row filters, and copy

This plan extends `VARIANT_TABLE_AND_JOB_UX_PLAN.md` after Slice 33. It focuses on making structural mismatch states understandable for non-technical merchants, especially when the system has current valid variant rows plus saved variants that no longer match the current options.

---

## Modern UX for Non-Technical Merchants — PRIMARY FOCUS

**This section establishes the emotional and psychological UX strategy as the primary architecture layer. The backend architecture is strong; now simplify and soften the merchant experience to maximize confidence, clarity, emotional safety, and cognitive simplicity.**

### The Core Shift: From System Language to Merchant Confidence

Non-technical merchants do NOT think about:
- States, lifecycle, stale active variants, operational blocking, classification, rebuild semantics
- Database concepts, occupancy keys, combination matrices, orphaned entities

They think about:
- "Can I continue selling?" ✅
- "Did I lose anything?" ✅ 
- "What should I click?" ✅
- "Is this safe?" ✅

**UX Goal: Merchants understand the page in 3 seconds.**

When opening the variants editor, they should instantly know:
1. Everything is safe (the dominant UX narrative)
2. What they should do next

The UI centers continuity and safety: active variants first, editing first, safety first, history second.

### Core UX Rules

1. **Merchant Momentum (The Primary UX Goal)**
   - Every screen, card, and state must answer: *"Can the merchant continue working immediately without hesitation?"*
   - Avoid creating hesitation, uncertainty, repair-thinking, fear of deletion, or system anxiety. The UI should always center confidence and continuity.

2. **Reduce Counts Shown Simultaneously**
   - Do NOT expose: current, archived, missing, total, review, possible, all at once.
   - Show **ONE primary number** (e.g., "243 active variants") and **ONE secondary contextual number** (e.g., "163 saved variants saved").
   - Avoid dashboard-like tables or metric walls.

3. **Fully Eliminate 'Review' from Merchant-Facing UX**
   - The word "Review" psychologically implies responsibility, unfinished work, and pending issues.
   - Replace with "Older variants saved. You can manage these anytime." and a calm, optional `[View]` button.

4. **Use Empty Space Aggressively**
   - Keep the design spacious, breathable, and focused. Do not try to fill every area with helper texts, metrics, or badges. Sometimes a simple success indicator and main CTA is enough.

5. **Collapse Secondary Information on Mobile**
   - Mobile merchants want to edit, publish, manage inventory, and change prices. They do not want lifecycle management.
   - Collapse the saved variants card, archived info, advanced tools, and status explanations automatically on mobile under a simple expander.

6. **Visually Detached Advanced Tools**
   - Power-user features (such as recreating variants or deep repair settings) must be visually detached from everyday options.
   - Use aggressive spacing, horizontal dividers, subdued styling, or accordion sections to place these actions far from primary CTAs (like editing variants) so they never distract.

### Merchant-First Terminology

Replace internal/database language with natural, non-technical merchant language. We strictly standardize naming and eliminate technical jargon from the visible UI:

| Internal / Code Term | Visible UI Term | Purpose / Category | Avoid Exposing |
| --- | --- | --- | --- |
| `current` / Current rows | **Active variants** | Primary / Default view | row, rows, current |
| `stale_active` / Previous-option rows | **Older variants** | Historical view (preserved) | stale, orphaned, previous-option, mismatch |
| `archived` / Archived rows | **Archived variants** | History view (inactive) | archived rows, logical delete |
| `missing` / Missing combinations | **Create variants** | Setup actions | missing, matrix, combinations, template quota |
| `rebuild` / Rebuild from scratch | **Recreate variants** | Advanced actions | rebuild, scratch, generation job |
| `orphanedCount` / `removedCount` | **Older variants saved** | Informational count | orphaned, removed, structural review, stale count |

*Strict UI Copy Policy*: Fully eliminate the words **review**, **stale**, **orphaned**, **mismatch**, **previous-option**, **structural**, and **rows** from all user-facing copy. Frame historical information optionally, letting merchants feel everything is safe and saved.

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

Replace warning banners and technical panels with clean workflow cards.

**Before (Warning-Heavy, Technical):**
```
⚠️ Product options changed

243 current variant rows are ready to edit.
163 rows from previous options no longer match your current options.
0 new combinations needed.

[Archive saved variants and recreate variants]  [View older]
```

**Modern (Clean, Confidence-Building, Calm):**
```
✅ Variants Ready

243 active variants ready to edit.

[Continue editing]


ℹ️ Saved Variants

163 saved variants saved. You can manage these anytime.

[View]
```

**Key Differences:**
- Lead with active/current (the safe, normal, default path).
- Use ✅ checkmark instead of warning ⚠️ sign.
- "Your Variants Are Ready" is the headline (positive psychological framing).
- The secondary card is calm, informational, and optional (uses "Older variants saved" and `[View]` instead of "Review").
- No mentions of "mismatch", "previous-option", "rows", or "stale".
- Empty space is prioritized to avoid information overload.

### Visual Hierarchy for Manage Variants Page

```
┌─────────────────────────────────────────┐
│  ✅ Variants Ready                      │  ← Largest, most prominent
│                                          │
│  243 active variants                    │  ← Primary number, clear purpose
│                                          │
│  [Edit variants]                        │  ← Primary action, simple CTA
└─────────────────────────────────────────┘

┌─────────────────────────────────────────┐
│  ℹ️ Saved Variants                 │  ← Secondary, informational card
│                                          │
│  163 saved variants saved.              │  ← Secondary count
│  You can manage these anytime.          │
│                                          │
│  [View]                                  │  ← Optional action (calm view)
└─────────────────────────────────────────┘

  ⋯ Advanced tools [expand]               ← Hidden by default, collapsed
```

This layout says: "Everything is safe. Continue working. Advanced options are available if needed."

### What to Hide (Keep Out of Merchant View)

Merchants should NEVER see or need to understand:

- `stale` / `stale_active` state
- `orphaned` or `orphanedCount` / `removedCount`
- `occupancy keys` or `combination keys`
- `rebuild semantics` or "rebuild from scratch"
- `lifecycle_status`
- `operational state` or `active_job`
- `incomplete_draft`
- `structural state` classifications
- Database metrics or internal counts
- Repair-heavy language, repair flows, or review workflows by default

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
[View]
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

✅ Variants Ready (243)
163 saved variants

[Continue editing]
```

**User clicks [More...]:**
```
✅ Variants Ready (243)
[Edit variants]

ℹ️ Older Variants (163)
[View these variants]

⚙️ Advanced tools
├─ Recreate variants from current options
├─ Archive saved variants
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

**Situation: Current + saved variants**
```
✅ 243 Active Variants

Ready to edit and sell.

ℹ️ 163 Older Variants

Saved from your previous options. Manage anytime.

[View]
```

**Situation: Missing variants (Setup)**
```
✨ Finish Setup

12 variants are ready to create to match your options.

[Create variants]

Saved variants:
[163 saved variants]
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
243 active / 163 saved for history
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

> All 243 active variants are ready. 163 saved variants from previous option setups are saved for history.

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
- Product publish status and variant structure status are separate. Storefront visibility still depends on published snapshots, not raw draft structure.

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
Archive or view history if needed.
```

Or even simpler, just gray it out silently with light hover help.

---

## Copy Translation Map — Modern Merchant Language

Translate technical/internal language into natural, confidence-building merchant language.

| Technical/Internal | Modern Merchant Language | Example Usage | Tone |
| --- | --- | --- | --- |
| Orphaned variants | Older variants | "163 saved variants" | Neutral, historical |
| Removed / Removed -163 | Older variants saved | "163 saved variants saved" | Reassuring |
| Structural changes detected | Your options changed | "You changed your product options" | Factual |
| Missing combinations / Missing rows | Variants to create | "Create 12 variants" | Action-oriented |
| Generate missing variants | Create variants | "Create variants" | Clear, setup-focused |
| Rebuild from scratch | Recreate variants | "Recreate variants" | Advanced, clear |
| Show archived | Row view or Include history | Segmented: "Active / Older / Archived / All" | Navigation, not technical |
| Combination matrix | (never expose) | Don't use | — |
| Older variants saved | Older variants | "163 saved variants saved" | Optional, calm |
| Occupancy / occupancy keys | (never expose) | Use variant, option, combination as needed | — |
| Stale active / Previous-option | Older variant | Badge/card: "Older variant" | Neutral |
| Lifecycle status | (never expose; use context) | "Active variant", "Archived variant" | Context-specific |
| Orphaned count | (never expose) | Show count only: "163 saved variants" | Count, no jargon |

**Rules:**
- Keep backend/test names technical where appropriate.
- User-facing copy must ALWAYS use merchant language from this table.
- User-facing copy must ALWAYS be in `admin-copy.ts` i18n pattern.
- Locale dictionaries should inherit safely through existing fallback/spread behavior.
- English source copy must be complete and natural-sounding.

---

## Target UX — Modern, Calm, Confidence-Building

### Manage Variants Top State (Complete Scenario)

**Scenario: Current variants are complete, saved variants exist but are not blocking.**

```
┌──────────────────────────────────────┐
│ ✅ Variants Ready           │
│                                       │
│ 243 active variants ready to edit.    │
│                                       │
│ [Edit Variants]                       │
└──────────────────────────────────────┘

┌──────────────────────────────────────┐
│ ℹ️ Saved Variants               │
│                                       │
│ 163 saved variants saved.             │
│ You can manage these anytime.         │
│                                       │
│ [View]                                │
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

[Archive saved variants and recreate variants] [View older]
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

**Before (Dashboard / Metric Table):**
```
| Metric | Count |
| --- | --- |
| Current | 243 |
| Missing | 0 |
| Previous-option | 163 |
| Archived | 2 |
| Total | 408 |
```

**Modern (Spacious, Focus on Continuity):**
- UI hides the counts dashboard entirely.
- Large Primary Number: **243 active variants** (on the main card).
- Secondary Contextual Number: **163 saved variants saved** (on the secondary view card).
- Archived count (2) is hidden from primary view and only visible inside the `[View]` historical panel.
- On Mobile: Only the active card is visible by default. The saved variants section collapses.

```
✅ Variants Ready
243 active variants

[Edit variants]
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
ℹ️ Saved Variants
163 saved variants

These are saved for history.
You can manage them anytime.
```

Smaller, softer, reassuring, not alarm-inducing.

### Studio Footer CTAs — Simplified

**Situation: Missing variants = 0, Older variants > 0**

Old (confusing):
```
[Generate missing variants (disabled)] [View older] [Rebuild]
```

Modern (clear):
```
[Continue Editing] [View Older]
```

Repair tools are hidden under ⋯ Advanced.

**Situation: Missing variants > 0**

Old:
```
[Create variants] [View older] [Rebuild]
```

Modern:
```
[Create Variants]

Create 12 variants to match your options.

[View Older] (secondary)
```

Simple, task-focused, no repair language.

---

## Safe Action Hierarchy — Modern Version

Every action should communicate confidence and next step through placement and visual weight.

| Action Type | Example | Visual Placement | Tone | Accessibility |
| --- | --- | --- | --- | --- |
| **Normal editing** | Save variant, change price | Table/inline controls | Default, unstressed | Full interactivity |
| **Safe creation** | Create missing variants | Primary button (top card) | Confident, positive | Large target, clear text |
| **Navigation/optional** | View older, view options | Secondary button | Calm, optional | Available but not forced |
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
| Current only | Neutral | Everything is ready | None; keep editing | View options | Advanced hidden |
| Missing only | Action required | Setup is incomplete | **Create variants** | View options | Recreate variants collapsed |
| Previous-option rows only | Non-blocking warning | Active variants ready, saved variants saved | **View older** | Keep editing active variants | Archive saved variants and recreate variants |
| Missing + saved variants | Action required | Finish setup first, then view saved variants | **Create variants** | View older | Recreate variants collapsed |
| Archived only included | Informational | User chose to view history | Continue editing active variants or switch view | Restore selected archived variant when safe | None |
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
- `View older` becomes primary only when there are no missing rows but saved variants exist.
- Repair/rebuild actions are **tertiary or hidden** on the everyday table.
- Publish and row-review CTAs remain separate concerns.

---

## Variant View & Filtering UX

Replace the single checkbox with a clean segmented selector:

- `Active variants` (Primary/default)
- `Older variants` (Historical)
- `Archived variants` (History)

Counts are displayed cleanly on the tabs:
- `Active (243)`
- `Older (163)`
- `Archived (2)`

Behavior:
- **Active variants**: Shows only variants that match the current options setup and are active.
- **Older variants**: Shows active variants from previous options (no longer matching).
- **Archived variants**: Shows archived/deleted variants.
- The default view is always **Active variants** when entering the editor.
- The word "row", "stale", "mismatch", "review", or "warning" is never used on tab labels or filters.
- Do not use `Older` or `Previous-option rows` in the tab label. Stick strictly to `Older variants`.

If full classification cannot be delivered immediately, phase it:

1. Rename `Show archived` to `Include archived rows`.
2. Add helper text: `Archived rows appear together with active rows.`
3. Later replace with segmented row views once row classification is available.

---

## Table Presentation

Variants should show clear status badges:

Active variant badge:
- `Active`
- Positive Green/neutral tone.

Older variant badge:
- `Older`
- Soft neutral/gray tone (must NOT use cautionary/warning colors like amber or yellow).
- Helper tooltip: `Saved from your previous options.`

Archived variant badge:
- `Archived`
- Muted gray tone.
- Helper tooltip: `Kept for history.`

Avoid mixing different variant types in a single view without grouping or explicit badges. If a combined view is active, group them under clear headings:
- **Active variants**
- **Older variants**
- **Archived variants**

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
  - `older` if not.
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

| State | Severity Level | Meaning | UI Treatment | CTA Weight | Badge Tone | Visual Tone |
| --- | --- | --- | --- | --- | --- | --- |
| `current` / active | Neutral | Active; ready to edit | Quiet, spacious | N/A | None or soft green | Positive (Green) |
| `missing_variants` | **Action required** | Setup is incomplete | Setup/Setup guidance card | **Primary** | Amber | Setup guidance (Amber) |
| `stale_active` (older) | **Informational** | Saved from previous options | Informational card | N/A / Low | Soft gray / neutral | Neutral-soft (Gray) |
| `archived` | Informational | Kept for history | Muted styling | N/A | Gray | Muted (Gray) |
| `failed_job` | **Error** | Last operation failed | Error card/panel | Tertiary (retry) | Red | Error (Red) |
| `active_job` | Processing | Background job running | Progress indicator | N/A (blocked) | Blue | Processing (Blue) |

**Severity Determines:**
- Color intensity and tone (saved variants must use soft grays, avoiding cautionary amber/yellow)
- Card vs badge presentation
- CTA prominence and spacing
- Whether actions block editing

**Continuity and Momentum Philosophy:**

Most critical: Older variants (`stale_active`) are **completely non-blocking** and **informational**.
- Active variants remain fully editable.
- Recreating variants is not forced.
- Merchant can continue editing and selling immediately (preserving merchant momentum).
- Historical viewing is optional and quiet.

This keeps merchant confidence high, prevents "system is broken" feelings, and removes cautionary alarms from normal flows.

---

## Naming Convention Refinement

Standardize terminology around merchant concepts, shortening names deeper in the flow to create a premium-feeling interface rhythm.

| Component / Context | Recommended Term | Purpose / Category | Avoid Exposing |
| --- | --- | --- | --- |
| **Main Card** | **Variants Ready** | Primary status summary card | Active variants ready, current rows |
| **Secondary Card** | **Saved Variants** | Preserved older variants card | Older variants saved, review, mismatch |
| **Tab / Filter** | **Older** | Historical selector tab | Older variants, stale, previous-options |
| **Badge / Label** | **Older** | Historical inline badge | Stale, needs review, orphaned |
| **History View / Tab** | **Archived** | Muted archived tab | Archived variants, deleted, logical delete |
| **Setup Button** | **Create variants** | Primary setup button (Setup) | Generate combinations, create missing |
| **Advanced Button** | **Recreate variants** | Advanced settings action | Rebuild from scratch, regenerate all |
| **Secondary Action** | **View** | Calm secondary action button | Review, fix, repair, verify |

**Terminology Forbidden List for Merchant UI:**
❌ `review` / `needs review`
❌ `stale`
❌ `orphaned`
❌ `mismatch` / `structural mismatch`
❌ `previous-option`
❌ `structural`
❌ `row` / `rows`

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
- Update English source copy in [admin-copy.ts](file:///d:/Github/muhsinmuhsy/PX/PX-F/lib/catalog/admin-copy.ts).
- Replace primary UI language:
  - `Orphaned variants` -> `Older variants`
  - `Removed` -> `Older variants saved`
  - `Show archived` -> Segmented tab navigation (`Active`, `Older`, `Archived`)
- Add helper copy explaining that saved variants are kept for history.
- Update Studio, Manage Variants, and Maintenance copy to eliminate "review", "stale", "mismatch", and "rows".
- Add/adjust i18n copy tests.

Acceptance:
- A non-technical user can tell that active variants are safe and saved variants are preserved.

### Phase 2: Clear Status Summary Cards

Goal:
- Replace developer-style metrics with merchant status summaries.

Tasks:
- Add a top summary card for Manage Variants:
  - Active variants ready (prominent number)
  - Variants to create (if missing)
  - Older variants saved (informational card with `[View]` button, hidden on mobile by default)
- Drive CTAs strictly from the Recommended Next Action Map.
- Update Studio right panel to use `Older` and `Saved` language.
- Hide all advanced repair metrics.

Acceptance:
- Main screen shows confidence, clear counts, and lets the user continue editing immediately.

### Phase 3: Variant View Filter

Goal:
- Make variant visibility obvious.

Tasks:
- Replace the `Show archived` checkbox with a segmented tab selector:
  - `Active variants`
  - `Older variants`
  - `Archived variants`
- Persist tab selection only locally unless product requirements need URL query state.

Acceptance:
- Users can toggle views calmly, understanding which variants match current options and which are historical.

### Phase 4: Recreate Flow Refinement

Goal:
- Make advanced recreating actions safe and visually detached.

Tasks:
- Rename rebuild action to explain outcome:
  - `Recreate variants`
- Add confirmation copy:
  - how many active variants stay,
  - how many saved variants will remain archived,
  - whether media bindings are affected.
- If `missingCount === 0`, do not show a disabled button as the primary action.
- Primary action for incomplete setup state is simply to continue editing. Recreate is hidden under advanced tools.
- Place all advanced recreating tools behind a visual divider or collapsible accordion far from primary CTAs to prevent accidental clicks.

Acceptance:
- Users understand history is preserved, and advanced tools do not clutter the page.

### Phase 5: Per-Variant Classification

Goal:
- Show status on every variant.

Tasks:
- Add a reusable classification helper.
- Add reusable editability helpers in `variant-capabilities.ts`.
- Feed status into `VariantIdentityCell` or `VariantTableRow`.
- Add badges:
  - `Active`
  - `Older`
  - `Archived`
- Group variants under clear headings if combined view is active.
- Keep selection/bulk updates limited to editable active variants by default.

Acceptance:
- Older and archived variants are labeled clearly and edit capabilities are locked safely with friendly tooltips.

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
  - summary for Active + Older variants,
  - variant view segmented selector tabs,
  - default view is `Active variants` when older or archived variants exist,
  - advanced maintenance panel is visually detached and secondary.

- `variant-structure-studio.test.tsx`
  - incomplete setup state does not show a disabled generate button,
  - Studio right panel says `Older variants saved` (or simply `Older`), not `Removed`,
  - incomplete setup state still shows create-variants primary action.

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
| `incomplete_draft` | No | No | No | Setup card | Primary action enabled |
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
| Rebuild action (safe flow) | Hidden | "Not relevant; no saved variants exist" | Not rendered; not in tab order |
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

## Progressive Prominence Reduction (Preserving Focus)

To prevent structural anxiety and preserve merchant focus, the visibility and prominence of historical data is reduced progressively as the merchant moves deeper into their editing flow:

1. **Entry Screen (High-Level Summary)**:
   - Older variants saved are presented in a quiet, secondary card (`ℹ️ Saved Variants`).
   - The card collapses automatically on mobile.
2. **Variant Table (Working Space)**:
   - The card disappears.
   - Historical variants are represented only as clean filter tabs (*Active (243)*, *Older (163)*, *Archived (2)*).
   - Only the selected tab's variants are visible (default is Active).
3. **Individual Editing Flow**:
   - The history is completely hidden. The merchant is focused 100% on editing and publishing active variants.

---

## Animation & Motion Philosophy

Modern, premium merchant software (like Shopify, Notion, or Linear) relies on motion design for visual continuity and cognitive simplicity. The frontend should implement:

- **Soft Expand/Collapse**: Smooth transitions (e.g. 200ms ease-out) when expanding advanced tools or toggling accordion sections.
- **Optimistic UI Updates**: Immediate UI feedback (e.g. state changes to "Saved" or local count updates) while backend requests execute in the background.
- **Cross-Fade View Switching**: Soft 150ms opacity fades when toggling between filter tabs (*Active*, *Older*, *Archived*) to make list changes feel continuous rather than abrupt.
- **Skeleton Loaders**: Quiet, static background-color pulsing skeletons instead of alarming spinners, preserving merchant momentum during reads.
- **Visual Save Confirmations**: Micro-animations on inline row success badges (a soft green flash or check animation) to build high merchant confidence.

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

- The UI never requires users to understand `orphaned`, `matrix`, `snapshot`, `combination_key`, or `lifecycle_status`.
- The first message in Manage Variants explains:
  - active variants,
  - saved variants,
  - next action.
- Default table view is `Current` when rows needing review or archived rows exist.
- `Older variants` and `Archived` remain separate states internally.
- Studio and Manage Variants use the same merchant-calm language for the same state.
- A checked control never implies “show only archived” when it actually means “include archived.”
- Destructive/advanced actions are clearly separated from everyday editing.
- Empty, loading, current, missing, needs-review, archived, and failed states are visually distinct.
- Current rows stay editable when saved variants need review unless a real backend safety rule blocks editing.
- Row editability is centralized and testable; components do not invent their own rules.
- **Primary CTAs follow the Recommended Next Action Map** — state → action mapping is deterministic and testable.
- Severity levels determine visual treatment: color, banner type, CTA weight, whether actions are blocked.
- Non-blocking warnings (saved variants) do not prevent active variant editing or suggest system failure.
- Centralized selectors (`canEditRow()`, `canBulkEditRows()`, etc.) are the single source of truth for permissions.
- Tests verify that all components use centralized selectors; ad hoc permission logic is forbidden.
- Naming is consistent by context: tabs use "Active variants" / "Older variants" / "Archived variants", buttons use "View", labels use "Older" / "Active" / "Archived".
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

✅ **Merchant Momentum First**: Every page and state preserves momentum; merchants can edit and save without hesitation or fear of structural deletion.

✅ **Merchants understand the entire page in 3 seconds** without reading explanations or grasping system concepts.

✅ **First visible content communicates two things only:**
1. "Everything is safe ✅"
2. "What to do next"

✅ **Language is always merchant-natural, never system-technical:**
- ✅ Use: "243 active variants", "Older variants saved", "Create variants"
- ❌ Never expose: `review`, `stale`, `orphaned`, `mismatch`, `previous-option`, `structural`, `row`, `rows`, `matrix`, `occupancy`

✅ **Visual tone is calm and confidence-building**, not cluttered or alarming:
- ✅ Checkmark icons (✅), informational cards (ℹ️), soft gray neutral tones for saved variants
- ❌ Setup guidance cards (⚠️) or amber cautionary colors for saved variants (reserve amber warning only for actual missing setup combinations)

✅ **Normal editing dominates the screen** — merchants feel: "Everything is okay. Keep working."
- Primary visible actions: [Edit], [Continue], [Create variants]
- Secondary actions: [View], [Details], [See options] (fully eliminate "Review" button text)
- Advanced/destructive actions: Visually detached far from main flow, hidden under [⋯ Advanced Tools]

✅ **Advanced tools are hidden and visually detached** — never place recreate/advanced actions near primary editing CTAs; use spacers, lines, or accordions.

✅ **Text density is reduced 40–50%**:
- Replace dense paragraphs with short info cards
- Replace metric tables with single primary count and single secondary count
- Use progressive disclosure for complex information

✅ **Mobile-first simplicity** — all complexity (including saved variants card and advanced tools) collapses automatically on mobile.

✅ **Modern card-based layout** (Shopify-like, not admin-dashboard):
- Safe/ready state in large, prominent card
- Secondary info in smaller soft-neutral card
- Advanced options completely hidden/collapsed

✅ **Terminology is consistent in merchant UI:**
- ✅ Always: "243 active variants", "Older variants", "Create variants"
- ❌ Never: "243 current variant rows", "rows needing review", "generate"

✅ **Empty, error, and processing states are visually distinct but not alarming** — use progress indicators, not error language.

### Engineering Architecture (Hidden But Fully Enforced)

✅ The UI never exposes to merchants: `orphaned`, `matrix`, `snapshot`, `combination_key`, `lifecycle_status`, `operational state`, `stale_active`, `occupancy`.

✅ Default table view is `Active variants` when older/archived variants exist.

✅ `Older variants` and `Archived variants` remain separate states internally (for filtering, bulk actions, analytics).

✅ Editability is centralized in `canEditRow()`, `canBulkEditRows()` selectors — **no ad hoc permission logic in components**.

✅ **Primary CTAs follow the Recommended Next Action Map** — state automatically determines the primary action.

✅ Severity and visual tone drive visual treatment (color, prominence, CTA weight, blocking behavior).

✅ Older variants (`stale_active`) are **always non-blocking** — merchants can edit active variants even with saved variants present.

✅ Naming is consistent by context:
- Tabs/filters: "Active variants", "Older variants", "Archived variants"
- Inline badges/labels: "Active", "Older", "Archived"
- Tooltips/help: "Saved from your previous options"

✅ **Visible UX is simple; hidden architecture is strong** — engineering excellence is transparent through effortless merchant experience.

---

## Open Questions

1. Should `Older` include archived rows that no longer match current options, or should archived rows always stay only under `Archived`?
2. Should the default table view be `Current` when saved variants exist?
3. Should row-view state persist in URL query params for refresh/shareability?
4. Do we need a backend `row_status` projection for performance once products have thousands of variants?
5. Should stale active rows allow direct SKU/price/stock edits, or should they be review/archive-first only?

Recommended default answers:

- Default view must be `Current`.
- `All` should group current, needs-review, and archived rows.
- Archived rows should remain visually separate from active needs-review rows.
- Start with frontend classification; move to backend projection only if needed.
- Centralize editability and recommended-action decisions in helpers/selectors before spreading logic into UI components.
