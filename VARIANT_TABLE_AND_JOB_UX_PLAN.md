# Variant Table and Job UX Stabilization Plan

Status: Active implementation plan  
Created: 2026-05-24  
Scope: PX-F product editor variant management UX and state flow

Decision status:
- Implemented in Slice 27: centralized variant job controller, completed-job auto-dismiss, failed-job persistence, manual terminal-card dismissal, 1.5 second polling fallback, 10 second idle heartbeat, and safe existing-row editing during missing/orphaned attention states.
- Adopted: completed jobs auto-dismiss; failed jobs persist until manual dismissal.
- Adopted: one combined "Variant" identity column as the primary table layout.
- Adopted: explicit save with row-level confidence feedback before any autosave work.
- Adopted: repair/destructive operations move away from the everyday table flow.
- Adopted: selection-based bulk editing replaces always-visible filtered/page bulk editing.
- Adopted: centralize variant job state handling in a dedicated controller hook before adding more effects.
- Adopted: use "Variants" as the canonical user-facing term.
- Adopted: use "product versions" only as onboarding/helper language.
- Adopted: orphaned rows should allow most safe edits unless a direct backend integrity risk exists.
- Adopted: define explicit ownership for row drafts, table state, and refresh reconciliation before implementing row save feedback.
- Implemented in Slice 29: exact-variant media binder from the variant table using the existing product media `variant_id` binding contract.
- Implemented in Slice 30: explicit soft/hard variant refresh intent, synchronous draft ownership, hard-refresh discard confirmation, and server-row pagination without hidden dirty-state merging.

## Purpose

This plan captures the confirmed UX/state issues in the product variant editor and proposes a staged implementation path. The goal is to make the variant management page feel reliable, calm, and fast for non-technical shop managers while preserving the existing backend variant domain rules.

Primary purpose:
- Redesign the variant editor's UI/UX architecture and flow so a non-technical store owner or shop manager can understand it easily.
- The interface should feel modern, guided, and safe.
- The user should never need to understand the internal variant-generation system, job system, structure hashes, snapshots, or matrix logic to manage their product.

This is not only a visual polish task. The table and surrounding state messages must be redesigned so a non-technical user can answer these questions without thinking:

- What product version am I looking at?
- What can I safely edit right now?
- What needs my attention?
- What happens if I click this button?
- Did my change save?

## UX Architecture North Star

The variant area should be designed as a simple guided workflow:

1. Define what versions of the product exist.
2. Create any missing variant rows.
3. Manage selling details for each variant.
4. Attach images when needed.
5. Fix rare problems only when the system asks.

The UI should not feel like:
- a database table,
- a developer debug panel,
- a queue dashboard,
- a matrix editor,
- an operations console.

The UI should feel like:
- a product inventory workspace,
- a guided setup flow,
- a safe editing table,
- a modern ecommerce admin screen.

## Recommended Information Architecture

### Main User Flow

The variants section should be organized around the user's mental model, not the backend model.

Recommended visible order:

1. **Setup Product Versions**
   - Add options such as Size, Color, Material.
   - Add values such as Small, Medium, Blue, Green.
   - Explain this as "Tell us what versions of this product you sell."

2. **Create Variant Rows**
   - If rows are missing, show one clear action.
   - Main wording should be similar to "Create missing variant rows."
   - Existing rows should remain editable unless a real unsafe state exists.

3. **Manage Variant Details**
   - This is the main table.
   - Focus on image, variant identity, SKU, price, stock, and visibility.
   - This area is where most users spend their time.

4. **Images**
   - Allow product-level images and variant-specific images.
   - Variant-specific image binding should become available from the table later.

5. **Repair & Maintenance**
   - Hidden or visually secondary.
   - Contains rebuild, orphan repair, detached media, diagnostics, and advanced recovery.

### What The User Should See First

When the user opens the variants tab, the first screen should answer:

- Do I have variants?
- Are they ready to edit?
- Is anything waiting for me?
- What is the next safe button to click?

The page should not lead with low-level metrics, internal job terminology, or repair actions.

## Modern Flow Requirements

Every state should have:
- one clear headline,
- one plain-language explanation,
- one primary action,
- secondary actions only when useful,
- a clear indication of whether editing is safe.

Examples:

- "No product versions yet" -> "Add options like Size or Color."
- "3 new variant rows needed" -> "Create the missing rows. Your existing rows can still be edited."
- "Creating variant rows" -> "This usually takes a few seconds. You can stay on this page."
- "Rows are ready" -> "You can now edit price, stock, SKU, images, and visibility."
- "Needs attention" -> "Some rows no longer match the current options. Review repair options."

The user should not see phrases like "matrix is out of sync" as the primary message.

## Non-Technical User Design Principles

### Use Store-Language, Not System-Language

Avoid exposing backend or engineering terms as the main user-facing language.

Prefer:
- "Product versions" or "Variants" over "combination matrix".
- "Missing variant rows" over "missing combinations".
- "Needs attention" over "orphaned variants".
- "Updating variants" over "processing job".
- "Live updates" only as secondary status, not primary meaning.
- "Saved" / "Still saving" / "Could not save" over technical sync language.

System terms can stay in logs, tests, and code, but the UI should speak like a calm store assistant.

Terminology decision:
- Use "Variants" as the main product term.
- Use "product versions" only to explain the concept during onboarding or empty states.
- Example: "Variants are different versions of this product, like Blue / XL."
- After that explanation, use "Variants" consistently.

### Make the Page Tell the User What To Do Next

Every major state needs one clear next action:

- No options yet: "Add options like Size or Color."
- Options exist but no variants: "Create variant rows."
- New value added: "Create the new missing rows. Existing rows can still be edited."
- Job running: "Creating rows. You can wait here."
- Job completed: "Rows are ready."
- Save failed: "Try again" plus the exact row or field if possible.

Avoid placing several equal-looking buttons beside scary warnings. The primary next step should be visually obvious.

### Reduce Spreadsheet Anxiety

The current table asks the user to scan many plain cells and input boxes. The redesign should make the table feel like inventory management, not a raw database grid.

Rules:
- One row should read as one sellable version of the product.
- The variant identity should be visually grouped first.
- SKU, price, stock, status, and image should be secondary editable properties.
- Inputs should appear active only where editing is expected.
- Save/changed state should be visible at row level.
- The table should look like a curated inventory editor, not an Excel sheet.
- The most common fields should be easy to scan without reading every column header.

### Progressive Disclosure

Advanced or destructive tools should not compete with everyday editing.

Everyday actions:
- Change price.
- Change stock.
- Toggle visibility.
- Assign image.
- Save row.

Advanced actions:
- Rebuild all variants.
- Archive variant.
- Repair detached media.
- Resolve orphaned rows.

Advanced actions should be visually quieter and often behind a repair/advanced area.

### Calm Warning Hierarchy

Not every warning should freeze the page.

Use three levels:
- Notice: informative, no blocking.
- Needs attention: clear action recommended, safe editing continues.
- Blocked: user cannot safely continue until fixed.

Missing variant rows should usually be "Needs attention", not "Blocked".

## Confirmed Current Issues

### 1. Completed Variant Job Card Can Linger

Current behavior:
- `AdminProductEditForm` tracks `activeVariantJob`.
- When a job becomes `completed`, the form refreshes the authoritative snapshot, media, and snapshots, then shows a success toast.
- The local `activeVariantJob` state is not explicitly cleared after completion.
- `ProductVariantsSection` renders `VariantJobPanel` whenever `activeVariantJob` is not null.

User impact:
- The completed job card can stay visible and feel stuck.
- The UI appears to update only after another user action triggers a fresh snapshot reconciliation.

Recommended behavior:
- Completed jobs should remain visible only briefly as success feedback.
- The card should support manual dismissal.
- After a short timeout, the card should clear automatically unless a newer job has started.
- Failed jobs should not auto-dismiss. A failed card contains actionable information and should remain visible until the user retries or dismisses it manually.

Primary files:
- `PX-F/app/components/admin-product-edit-form.tsx`
- `PX-F/app/components/product-editor/product-variants-section.tsx`

### 2. Fallback Polling Feels Too Slow

Current behavior:
- SSE is preferred for live variant job events.
- If SSE cannot connect, the frontend falls back to polling `getVariantJob`.
- The fallback interval is currently 3000 ms.

User impact:
- When SSE is unavailable, the job HUD can feel frozen.
- A user may click Generate and wait too long before seeing reassuring progress.

Recommended behavior:
- Reduce active-job fallback polling to 1500 ms.
- Keep the live/polling badge because it is useful operational feedback.
- Avoid aggressive polling when no active job exists.

Primary file:
- `PX-F/app/components/admin-product-edit-form.tsx`

### 3. No Idle Background Active-Job Check

Current behavior:
- Active jobs are discovered on initial snapshot reconciliation and when user actions call `reconcileSnapshot`.
- If another tab or background process starts a job while this page is idle, the visible page may not notice.

User impact:
- Multi-tab usage feels inconsistent.
- The queue HUD can fail to appear until the user performs another action.

Recommended behavior:
- Add a lightweight idle heartbeat only when no active job is visible.
- Every 10 seconds, call `getActiveVariantJob(productId)`.
- If a queued/running job is found, set `activeVariantJob` and let the existing SSE/polling flow take over.
- Pause this heartbeat when a job is already active, when the product id is invalid, or when the component is unmounted.
- Implement this through a single job controller hook rather than more scattered effects.
- The controller must prevent duplicate SSE subscriptions, stale polling updates, and dismissed/completed job reappearance.

Primary file:
- `PX-F/app/components/admin-product-edit-form.tsx`

Recommended abstraction:
- `PX-F/app/components/product-editor/use-variant-job-controller.ts`

Responsibilities:
- active job discovery,
- SSE subscription,
- fallback polling,
- idle heartbeat,
- completion handling,
- failure handling,
- manual dismissal,
- stale job reconciliation,
- cleanup on unmount.

### 4. Variant Table Locks Too Much

Current behavior:
- `manageVariantsLocked` becomes true when combinations are not ready, missing combinations exist, or orphaned combinations exist.
- The lock disables SKU, price, stock, visibility, save, reset, archive, and restore controls in existing rows.

User impact:
- A manager who adds a new value, such as Size XXL, cannot keep editing existing variants until missing combinations are generated.
- The system blocks normal work for a structural warning.

Recommended behavior:
- Split the current lock into separate concepts:
  - `structureActionsLocked`: blocks risky option/value/rebuild operations.
  - `variantEditingWarned`: shows warning state but allows edits on existing non-archived variants.
  - `variantEditingLocked`: only true for active jobs, initialization, hard invalid states, archived rows, or permission/security constraints.
- Missing combinations should warn and guide the user to generate rows, not block editing existing rows.
- Orphaned variants should use a stronger "Needs attention" warning, but should still allow most safe edits unless backend invariants require blocking.
- Safe edits include stock, visibility, media, pricing, and SKU.
- Blocking should be reserved for active rebuild/generate jobs, hard invalid states, permission/security failures, and operations that would damage structure integrity.

Primary files:
- `PX-F/app/components/product-editor/product-variants-section.tsx`
- `PX-F/app/components/product-editor/variant-table-row.tsx`
- `PX-F/app/components/product-editor/use-product-variant-actions.ts`

### 5. Table Is Hard To Understand For Non-Technical Users

Current behavior:
- Variant option values are shown as plain text cells.
- The table exposes many editable fields at the same visual weight.
- Bulk edit is always visible and applies to filtered/page variants, which can feel risky.
- Important system states are described with technical language such as combinations, structure, sync, jobs, polling, and orphaned variants.
- The visual relationship between variant identity, price, stock, status, and media is weak.

User impact:
- A shop manager may not immediately understand what each row represents.
- Users may fear changing the wrong rows.
- Warnings can feel like errors even when the system is only asking them to create new rows.
- The page can feel like an admin database instead of a product editing workflow.

Recommended behavior:
- Redesign the table as a guided inventory editor.
- Make the first column a strong "Variant" identity column with color/image/value chips.
- Use plain-language state messages and clear next actions.
- Hide or soften advanced repair controls until needed.
- Make bulk actions selection-based so the user can see exactly what will change.
- Add row-level save state so the user knows what happened.
- Extract variant identity rendering into a reusable component so the table does not accumulate fragile display logic.

Recommended abstraction:
- `PX-F/app/components/product-editor/variant-identity-cell.tsx`

Responsibilities:
- render ordered option/value selections,
- show color/image/text chips consistently,
- handle archived and attention states,
- avoid duplicating chip logic across table, bulk bar, and media binder flows.

## Variant Table UX Upgrade Plan

### Phase 0: UX Language and State Map

Goal:
- Make the entire variants area understandable before moving pixels around.
- Define the UI architecture and user flow before implementing component-level changes.

Tasks:
- Audit all user-facing copy in the variant editor.
- Replace technical labels with shop-manager language where possible.
- Map the variants tab into the main user flow:
  - setup product versions,
  - create missing rows,
  - manage variant details,
  - images,
  - repair and maintenance.
- Define the allowed visible states:
  - no options,
  - options incomplete,
  - ready to create rows,
  - rows current,
  - new rows needed,
  - updating rows,
  - update complete,
  - needs repair,
  - failed.
- For each state, define:
  - headline,
  - short explanation,
  - primary action,
  - secondary action,
  - whether row editing is allowed.
- Keep engineering terms out of the main UI unless there is no clearer alternative.
- Decide which panels are everyday panels and which panels are advanced/repair panels.

Expected result:
- The user can understand the page flow before interacting with the table.
- Future implementation work has a clear UX architecture instead of scattered UI patches.

### Phase 1: Reliability and State Feedback

Goal:
- Make the current UI trustworthy before redesigning the table.

Tasks:
- Create `useVariantJobController(productId)` or equivalent before adding new job effects.
- Add completed-job auto-dismiss after 5 seconds.
- Add a close/dismiss button to `VariantJobPanel` for completed and failed jobs.
- Clear `activeVariantJob` only when the cleared job id still matches the current job id.
- Reduce active fallback polling from 3000 ms to 1500 ms.
- Add idle active-job heartbeat every 10 seconds when no active job exists.
- Keep failed job cards persistent until manual dismiss or retry.
- Add or update tests for:
  - completed job clears after refresh and timeout,
  - dismiss button clears the panel,
  - failed job does not auto-dismiss,
  - polling fallback uses the shorter interval,
  - idle heartbeat discovers a queued/running job.

Expected result:
- The queue card feels live.
- Completed feedback is visible but does not become stale UI.
- Multi-tab/background jobs become visible without forcing the user to touch the page.

### Phase 2: Unlock Safe Existing Variant Editing

Goal:
- Let managers continue useful work even when the structure has missing combinations.

Tasks:
- Replace the broad `manageVariantsLocked` behavior with narrower lock states.
- Keep structural warning card visible when combinations are missing.
- Keep Generate Missing Combinations prominent.
- Allow editing existing active variant fields during missing-combination state:
  - SKU,
  - price override,
  - stock quantity,
  - visibility,
  - save/reset.
- Continue blocking edits during active variant jobs.
- Continue blocking archived-row direct edits unless restoring is explicitly allowed.
- Update tests that currently expect all row controls to disable when missing combinations exist.

Expected result:
- The UI warns without freezing the manager's workflow.

### Phase 3: Premium Variant Row Presentation

Goal:
- Reduce spreadsheet fatigue and make rows easier to scan.
- Make the table understandable for a store manager who has never heard the word "matrix".

Tasks:
- Redesign the table around these columns:
  - selection checkbox,
  - variant identity,
  - image,
  - SKU,
  - price,
  - stock,
  - visibility,
  - save/status actions.
- Use `ValueVisual` for option values when the option display type is `color` or `image`.
- Render option values as compact chips:
  - color/image visual,
  - value label,
  - muted fallback for text-only options.
- Build this through `VariantIdentityCell` so ordering, chip rendering, and archived/attention styling remain centralized.
- Group the option chips into a single "Variant" visual area if the table becomes too wide.
- Do not show every option as its own equal-weight column when that makes the table feel like a spreadsheet.
- Prefer a readable identity stack such as "Blue / XL" with chips inside one strong first column.
- Keep table density professional:
  - compact row height,
  - stable column widths,
  - no layout shift on hover or edit state.
- Use clearer stock visuals:
  - normal stock,
  - low stock,
  - out of stock.
- Use status controls that read as product visibility, not database booleans.
- Add responsive behavior for narrow screens:
  - horizontal scroll is acceptable,
  - row content must not overlap,
  - buttons and inputs must keep stable dimensions.

Expected result:
- Variant combinations become visually scannable instead of plain text cells.
- A non-technical user can immediately see "this row is the Blue XL version" and then edit its selling details.

### Phase 4: Selection-Based Bulk Action Bar

Goal:
- Make batch updates explicit and precise.

Current capability:
- Bulk update already exists, but it applies to filtered/current-page variants.

Recommended behavior:
- Add row checkboxes.
- Show a sticky/floating bulk action bar only when one or more rows are selected.
- Bulk actions:
  - set price override,
  - set stock,
  - add stock delta,
  - set visibility,
  - clear selection.
- Replace the always-visible filter/page bulk editor with selection-based bulk editing.
- Filter/page-wide bulk actions are risky because users can forget a filter is active or misunderstand the scope.
- Add confirmation for large selections if needed.
- Show a human-readable selection summary, such as "6 variants selected".
- The bulk bar should never appear like a permanent form the user is expected to understand before selecting rows.

Expected result:
- Managers can confidently change many variants without editing rows one by one.

### Phase 5: Inline Save Feedback

Goal:
- Remove "did it save?" anxiety.

Recommended behavior:
- Preserve explicit save behavior and improve confidence:
  - changed fields get a subtle dirty state,
  - row save button becomes visually active,
  - after save, show a temporary row-level Saved check indicator.
- Consider a row status area with:
  - Unsaved,
  - Saving,
  - Saved,
  - Could not save.
- Do not implement autosave in this phase.
- Add unsaved-change protection before navigation, tab close, product switching, or snapshot switching once row-level dirty state exists.

Reasoning:
- Full autosave may require debounce, cancellation, conflict handling, and more careful validation UX.
- A row-level saved indicator gives most of the confidence without changing the data contract too much.
- Inventory edits are high-trust data; explicit save is safer until conflict and recovery behavior is mature.

Expected result:
- Users receive immediate, local confirmation after each row update.

State model:
- Row save state should be isolated from broad table-derived state as much as possible.
- Track row status as `pristine`, `dirty`, `saving`, `saved`, or `failed`.
- Pagination and snapshot refresh must not erase visible failed/dirty state unexpectedly.

Primary files:
- `PX-F/app/components/product-editor/variant-table-row.tsx`
- `PX-F/app/components/product-editor/product-variants-section.tsx`
- `PX-F/app/components/product-editor/use-product-variant-actions.ts`

### Phase 5A: Variant Editing State Ownership

Goal:
- Prevent dirty row state, pagination, background refreshes, and save feedback from becoming fragile.

Recommended ownership model:

- Row owns:
  - local field draft,
  - dirty state,
  - saving state,
  - saved state,
  - row-level save error.
- Table owns:
  - selected rows,
  - pagination,
  - filters,
  - visible sort/grouping state,
  - bulk action state.
- Shared reconciliation/controller layer owns:
  - merge rules between server refreshes and local dirty rows,
  - stale row handling,
  - hard refresh reset rules,
  - page refresh behavior.

Implementation guidance:
- Do not let every parent effect directly mutate row draft state.
- Do not let server refreshes blindly overwrite active dirty rows.
- Keep failed save state visible until the user edits again, retries, or performs a hard refresh.
- Keep `useVariantPagination` as a server-row loader only. Draft preservation belongs to the editor/table draft layer so refresh behavior stays explicit and testable.

### Phase 5B: Soft Refresh vs Hard Refresh

Status:
- Implemented in Slice 30.

Goal:
- Make background updates safe and predictable.

Soft refresh:
- Used for normal page refreshes, idle heartbeat results, snapshot reconciliation, and page pagination reloads.
- Merges server updates into clean rows.
- Preserves dirty local rows.
- Preserves failed save states.
- Preserves in-progress edits.
- Should not surprise the user by wiping unsaved values.

Hard refresh:
- Used after rebuild, repair, explicit discard, product switch, or user-confirmed reset.
- Discards local draft state.
- Reloads authoritative server state.
- Clears row-level save indicators unless a global failure remains.
- May be required when row identities are no longer valid.

Implementation guidance:
- Add explicit refresh intent where needed instead of treating every `reconcileSnapshot()` as the same kind of refresh.
- A hard refresh that will discard edits must warn the user if dirty rows exist.

### Phase 6: 1-Click Media Binder

Status:
- Implemented in Slice 29 for exact `variant_id` media binding.

Goal:
- Make variant media assignment fast from the variant table itself.

Existing foundation:
- Product media already supports `variant_id` binding.
- The media section already supports rebinding and unassigned media workflows.
- Backend support is exact-variant only today: media attaches to one concrete `variant_id`, or remains product-level/unassigned with `variant_id: null`.

Recommended behavior:
- Add a thumbnail cell to each variant row.
- Show the currently bound variant media if available.
- Clicking the thumbnail opens a compact media picker grid.
- Selecting a media item binds it to that exact variant row with one action by patching the media item's `variant_id`.
- Do not model "bind this image to all Blue variants" in the first implementation. That would require either repeated exact-row bindings in the frontend or a later backend media-to-option-value binding model.
- Show a saved indicator after binding.

Implementation note:
- This should come after Phases 1-5 because it crosses variant table state and media state.
- The implementation should reuse existing media APIs and state where possible.

Expected result:
- Assigning product photos to variants becomes a direct table workflow instead of a separate media-management task.

## Proposed Implementation Order

1. Phase 0: UX language and state map.
2. Phase 1: job HUD reliability and polling/heartbeat behavior.
3. Phase 2: unlock safe editing during missing-combination state.
4. Phase 5: row-level save feedback and unsaved-change protection.
5. Phase 5A/5B: editing state ownership and soft/hard refresh rules.
6. Phase 3: visual chips and row presentation.
7. Phase 4: selection-based bulk editing.
8. Phase 6: thumbnail media binder.

Reasoning:
- Save confidence affects user trust more than visual polish.
- Visual redesign should happen after the editing state model can clearly show dirty/saving/saved/failed rows.

## Design Acceptance Criteria

Before implementation is considered complete, the variants flow should pass these checks:

- A non-technical user can describe what the page is for in one sentence.
- The user can identify the next safe action within 5 seconds.
- Missing rows do not look like a catastrophic error.
- Repair actions do not visually compete with everyday editing.
- The table's first column clearly explains what product version each row represents.
- Save state is visible at row level.
- Bulk editing cannot happen accidentally through a hidden filter/page scope.
- Job/queue behavior feels like background progress, not an operations dashboard.
- Mobile and desktop layouts remain readable without overlapping text or controls.
- The main UI does not require understanding snapshots, hashes, SSE, polling, jobs, or combination matrices.

## System Fit Check

This plan has been checked against the current `PX-F` and `PX-B` architecture.

### Backend Support Already Exists

The backend already supports the major planned UX changes:

- Variant job creation:
  - `POST /catalog/admin/products/{product_id}/variant-jobs/generate-missing`
  - `POST /catalog/admin/products/{product_id}/variant-jobs/rebuild`
- Active job discovery:
  - `GET /catalog/admin/products/{product_id}/variant-jobs/active`
- Job status lookup:
  - `GET /catalog/admin/variant-jobs/{job_id}`
- Job event streaming:
  - `GET /catalog/admin/variant-jobs/{job_id}/events`
- Variant row editing:
  - `PATCH /catalog/admin/products/{product_id}/variants/{variant_id}`
- Variant archive and restore:
  - `DELETE /catalog/admin/products/{product_id}/variants/{variant_id}`
  - `POST /catalog/admin/products/{product_id}/variants/{variant_id}/restore`
- Media binding:
  - `PATCH /catalog/admin/products/{product_id}/media/{media_id}`
  - payload supports `variant_id` and `needs_variant_rebinding`
- Authoritative product structure state:
  - `GET /catalog/admin/products/{product_id}/authoritative-snapshot`
  - includes active job, structure hash, existing combination keys, snapshot state, and rebuild/repair data where available.

### Important System Observations

- Editing an existing variant row does not require the product structure guard. This supports the plan to allow safe row edits even when missing rows exist.
- Structural actions such as option/value mutation, generate missing rows, and rebuild do use structure guards. These should remain protected.
- The backend active-job endpoint returns active jobs only. A completed dismissed job should not be rediscovered by the idle heartbeat.
- Media is currently bound to exact variants through `variant_id`. Binding one image to all variants sharing a color/value is not a first-class backend concept today; the frontend would need to apply the same media choice across multiple exact variant rows or wait for a new backend model.
- Paginated variant loading already preserves local dirty edits while pages refresh. New row save-state work must integrate with this behavior carefully.
- Published storefront snapshots are real system concepts, but the everyday UI should not expose snapshot mechanics as the primary mental model.

### Frontend Work Required

The plan is mostly frontend architecture and UX work, not a backend rebuild.

Needed frontend additions:
- `useVariantJobController` for job discovery, SSE, polling fallback, completion, failure, dismissal, and cleanup.
- `VariantIdentityCell` for readable variant identity rendering.
- Row save-state model: `pristine`, `dirty`, `saving`, `saved`, `failed`.
- Unsaved-change exit protection.
- Selection-based bulk action state.
- Repair/maintenance panel separation.
- Plain-language copy revisions in `lib/catalog/admin-copy.ts`.

Possible backend additions later:
- Optional bulk variant update endpoint if selection-based bulk editing becomes slow with many rows.
- Optional media-to-option-value binding model if the product should support "bind this image to every Blue variant" as a native concept.

## Testing Strategy

Frontend unit/component tests:
- `PX-F/app/components/__tests__/admin-product-edit-form.test.tsx`
- `PX-F/app/components/product-editor/__tests__/product-variants-section.test.tsx`
- `PX-F/app/components/product-editor/__tests__/use-product-variant-actions.test.ts`

Recommended validation commands:

```powershell
cd PX-F
npm test
npm run lint
npm run typecheck
npm run build
```

Manual UX checks:
- Ask whether a non-technical user can tell what the next action is in every state without reading developer-like wording.
- Start a missing-variant generation job and verify the HUD appears immediately.
- Simulate SSE failure and verify polling updates within about 1.5 seconds.
- Let a job complete and verify the completed card can be dismissed and auto-clears.
- Force a job failure and verify the failed card remains visible until manual dismissal or retry.
- Open two tabs, start a job in one tab, and verify the other tab discovers it within about 10 seconds.
- Add a new option value and verify existing variants remain editable while missing combinations are shown.
- Edit a variant row and verify dirty/saving/saved/failed states are understandable.
- Try to navigate away with unsaved variant edits and verify the user is warned.
- Check table layout on desktop and mobile widths.

## Open Review Questions

No open review questions currently block the next frontend implementation.

## Resolved Review Questions

1. Completed job cards auto-dismiss after 5 seconds.
2. Failed job cards persist until manual dismissal or retry.
3. The main table uses one combined "Variant" identity column.
4. The first save-confidence milestone uses explicit save, not autosave.
5. Existing filter/page bulk update should be replaced by selection-based bulk editing.
6. Advanced repair actions should move into a separate repair/maintenance panel.
7. "Variants" is the canonical term; "product versions" is helper/onboarding language only.
8. Orphaned rows should allow safe edits unless there is a direct backend integrity risk.
9. Row draft ownership, table state ownership, and refresh reconciliation must be explicit before row save feedback is implemented.
10. Table media binding should use exact `variant_id` binding. Binding one image to all variants sharing a color/value is a future backend/product-model decision, not the next practical implementation.
