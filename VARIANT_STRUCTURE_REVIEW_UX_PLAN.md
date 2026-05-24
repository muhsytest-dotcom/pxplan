# Variant Structure Review UX Plan

Status: Proposed next UX hardening plan
Created: 2026-05-24
Scope: PX-F product editor Variant Studio, Manage Variants table, maintenance panel, row filters, and copy

This plan extends `VARIANT_TABLE_AND_JOB_UX_PLAN.md` after Slice 33. It focuses on making structural mismatch states understandable for non-technical merchants, especially when the system has current valid variant rows plus previous-option rows that no longer match the current options.

---

## Current Problem

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

## Merchant-Friendly State Model

The UI should translate backend states into three merchant-facing row groups, backed by explicit internal classification so the merchant labels do not blur important behavior.

Recommended internal row status model:

| Internal Status | Merchant Label | Meaning | Default Editability |
| --- | --- | --- | --- |
| `current_active` | Current | Active row matches the current option setup. | Editable |
| `stale_active` | Needs review | Active row exists but no longer matches current options. | Limited |
| `archived` | Archived | Row is archived and preserved for history. | Read-only by default |

These statuses should stay distinct for bulk actions, filtering, analytics, exports, and future permission rules even when the UI groups them visually.

Future state to keep separate:

| Future State | Merchant Label | Meaning |
| --- | --- | --- |
| `incomplete_draft` | New rows needed | Current options contain valid combinations that do not have variant rows yet. |

`incomplete_draft` is different from `stale_active`: missing rows mean new current combinations need to be created, while rows needing review mean previous-option rows already exist outside the current setup.

Editability must be explicit in implementation. Do not let each component rediscover this logic independently.

Recommended editability rules:

| Internal Status | SKU/Price/Stock | Visibility | Media Binding | Bulk Actions | Restore/Archive |
| --- | --- | --- | --- | --- | --- |
| `current_active` | Yes | Yes | Yes | Yes | Archive allowed |
| `stale_active` | Limited or yes when backend-safe | Limited | Limited | No by default | Archive allowed |
| `archived` | No | No | No | No | Restore when safe |
| `incomplete_draft` | N/A | N/A | N/A | N/A | Create missing rows |

If the product team wants stale active rows to remain editable, keep that choice explicit and tested. The default safe UI should prevent bulk edits from accidentally changing rows that no longer match the current option setup.

### 1. Current Rows

Meaning:
- Variant rows that match the current option setup.
- Safe for normal selling edits.

User-facing language:
- `Current variants`
- `Ready to edit`
- `Matches current options`

Allowed actions:
- Edit SKU, price, stock, visibility, image.
- Select rows for bulk updates.

### 2. Rows Needing Review

Meaning:
- Rows that exist but no longer match the current option setup.
- This maps to `orphanedCount` / invalid occupancy keys.
- Primary target is active rows outside the current setup (`stale_active`).
- Archived rows should remain visually separate by default, even if they also came from previous options.

User-facing language:
- `Rows needing review`
- `Rows from previous options`
- `No longer matches current options`

Label note:
- `Needs review` is action-oriented.
- `Previous-option rows` is more directly explanatory.
- Final tab/filter label should be usability-tested; if users find `Needs review` vague, prefer `Previous-option rows` as the visible tab and explain review in helper text.

Avoid as primary UI language:
- `Orphaned variants`
- `Removed`
- `Invalid combination`

Allowed actions:
- Review row details.
- Archive if active.
- Restore only if the row can safely become active under uniqueness rules.
- Rebuild/archive previous-option rows only through advanced confirmation.

### 3. Archived Rows

Meaning:
- Rows intentionally archived and kept for history.

User-facing language:
- `Archived variants`
- `Kept for history`

Allowed actions:
- View read-only by default.
- Restore when allowed.

---

## Copy Translation Map

Replace technical terms in visible UI.

| Current Text | Preferred Text |
| --- | --- |
| Orphaned variants | Rows needing review |
| Removed -163 | 163 previous-option rows need review |
| Structural changes detected | Product options changed |
| Missing combinations | New variant rows needed |
| Generate missing variants | Create missing rows |
| Rebuild from scratch | Archive previous-option rows and recreate variants |
| Show archived | Row view: Active / Needs review / Archived / All |
| Combination matrix | Product versions |

Notes:
- Keep backend/test names technical where appropriate.
- User-facing copy must use the existing `admin-copy.ts` i18n pattern.
- Locale dictionaries should inherit safely through existing fallback/spread behavior, but English source copy must be complete.

---

## Target UX

### Manage Variants Top State

When current rows are complete but previous-option rows exist:

```text
Product options changed

243 current variant rows are ready to edit.
163 rows from previous options no longer match your current options.

You can keep editing current rows. Review previous-option rows when you are ready.
```

Primary action:
- `Review rows needing review`

Secondary actions:
- `Edit options`
- `Open maintenance`

Advanced action:
- `Archive previous-option rows and recreate variants`

Rules:
- Do not show a scary repair panel by default.
- Do not lead with metric cards.
- Keep the normal table visible and editable.
- Keep advanced destructive tools visually secondary.
- Default table view must be `Current` whenever rows needing review or archived rows exist.
- Add a small `Why am I seeing this?` helper link near the review message.

Recommended helper:

```text
These rows were created using previous product options. They are preserved safely for history and can be reviewed or archived later.
```

Avoid making `Variants need review` the main page headline unless usability testing shows merchants understand it better. It can still work as a badge or secondary status, but `Product options changed` better frames the situation as a normal workflow event rather than a failure.

### Variant Studio Header

Current header should explain setup state, not internal mechanics.

Recommended:

```text
Product versions
Editing

243 current versions / 406 total rows / 163 need review
```

If there are previous-option rows:

```text
Product options changed
Some rows from previous options no longer match this setup. They are kept safely for review.
```

Preferred wording:

```text
Product options changed
Some rows were created from previous options. They are kept safely for review.
```

### Studio Right Panel

Replace:
- `Removed -163`

With:

```text
Needs review
163 previous-option rows
```

Add short helper:

```text
These rows were created from previous options and are not part of the current setup.
```

If rebuild preview is shown, use future-tense language:

```text
If you rebuild, these previous-option rows will be archived and new rows will be created from the current options.
```

### Studio Footer

When `missingCount === 0` and `orphanedCount > 0`:

- Do not show a large disabled `Generate missing variants` button.
- Primary action should be `Review rows needing review`.
- Secondary action should be `Close`.
- Advanced repair should stay behind `Show repair actions`.

When `missingCount > 0`:

- Primary action: `Create missing rows`
- Helper: `Existing rows will stay unchanged.`

When both missing and previous-option rows exist:

- Primary action: `Create missing rows`
- Secondary action: `Review rows needing review`
- Explain both states separately.

---

## Safe Action Hierarchy

Every action should communicate risk through visual weight and placement.

| Action Type | Example | Visual Treatment |
| --- | --- | --- |
| Everyday edit | Save row, change price, change stock, assign image | Normal table controls |
| Positive setup | Create missing rows | Primary action |
| Review/navigation | Review rows needing review, Edit options, Open maintenance | Neutral secondary action |
| Destructive/repair | Archive previous-option rows and recreate variants, rebuild | Danger tertiary, collapsed behind maintenance/repair |

Rules:
- Destructive/repair actions must never visually compete with row save, publish, or create-missing actions.
- Rebuild/archive actions should require clear confirmation copy and should stay outside the everyday editing path.
- When the product only has previous-option rows to review and no missing rows, the primary visible action should be review/navigation, not repair.
- The UI should make safe continuation obvious before showing cleanup tools.

---

## Recommended Next Action Map

The primary CTA should be derived from state, not chosen ad hoc in each component.

| State Combination | Merchant Meaning | Primary Action | Secondary Action | Advanced Action |
| --- | --- | --- | --- | --- |
| Current only | Everything is ready | None; keep editing | Edit options | Maintenance hidden |
| Missing only | Setup is incomplete | Create missing rows | Edit options | Rebuild hidden/collapsed |
| Previous-option rows only | Current rows are ready, previous rows need review | Review rows needing review | Keep editing current rows | Archive previous-option rows and recreate variants |
| Missing + previous-option rows | Finish current setup first, then review previous rows | Create missing rows | Review rows needing review | Rebuild collapsed |
| Archived only included | User chose to view history | Continue editing current rows or switch view | Restore selected archived row when safe | None |
| Active job running | Background update in progress | Wait/show progress | None | All structure actions disabled |
| Failed job | Last update failed | Retry or review error | Keep safe current edits if allowed | Repair collapsed |

Rules:
- `Create missing rows` outranks review when missing rows exist because it completes the current setup.
- `Review rows needing review` outranks repair when there are previous-option rows but no missing rows.
- Repair/rebuild actions should never become the primary CTA on the everyday table.
- Product publish CTAs should remain separate from row-review CTAs.

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
- Primary CTAs follow the Recommended Next Action Map.

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
