# Agent Handoff and Continuity

Last updated: 2026-05-24

Read this document first, then `00_START_HERE.md`, `WHAT-DONE.md`, and the architecture/design specs. Treat this file, `WHAT-DONE.md`, and the live code as the authoritative source of truth for the current state.

---

## 1. Latest Implementation Status (Slice 32 Structure Guard Payload Fix Complete)

The core product variant synchronization engine, background processing pipeline, multi-tenant boundaries, logical archiving/retention strategies, frontend/backend option deletion optimistic concurrency guard contracts, and the first three variant table UX modernization slices are **implemented and validated**.

### Completed Slices (Slices 1 to 25 & Guard Alignment):
- **Core Domain & Identity**: Canonical option combination keys (locale-independent sorted hashes) and active-only database-level uniqueness constraints.
- **Durable Progress Streams**: Server-Sent Events (SSE) background job tracking with monotonic sequence ordering and automatic client-side reconnection/API polling fallback.
- **Explosion & Quota Protections**: Destination-scoped target store tier policy validation (`Basic` vs. `Pro` limits) checking matrix limits on option apply, templates, copy actions, and background worker queues.
- **Decoupled Job Workers**: Process pool worker isolation from the REST API, cooperative job cancellation, dynamic error classification, and retry management.
- **Atomic Publication Boundaries**: Immutable structure snapshots Captured at job enqueuing, with explicit published version scoping isolating the public storefront from live admin drafts.
- **Internationalization (i18n)**: Backend-envelope error translation registry and frontend catalog dictionary mapping.
- **Observability HUD**: Variant job metrics panel, live/polling status telemetry, and active structure preview counts (`~` editing prefixes).
- **Logical Archiving / Retention Strategy (Slices 20-25)**:
  - Default soft-deletions (`is_archived = true` and `archived_at` timestamps) instead of cascade physical deletions.
  - Custom auditing log engine capturing the operator ID, timestamp, and operational reason for archiving/restoration.
  - Active-only variant index structure, allowing merchants to re-create active variations with previously archived SKUs/combinations.
  - Administrative restoration drawer, browse filters for archived variants, and strict inputs locking.
  - PostgreSQL-only test harness and env settings silencing legacy splitting warnings (`path_separator = os`) and eliminating the standard library `datetime.utcnow()` deprecation warnings.
- **Option Deletion Concurrency Guard Contract**:
  - Aligned frontend deletion API calls to append `expected_version` and `expected_hash` query parameters using the browser-native `URL` object wrapper.
  - Validated `structureGuard` presence and validity early, setting UI status to `refreshRequired` when the guard is missing or stale.
  - Verified and asserted behavior in unit testing (both positive route parameter verification and negative test cases verifying that missing/invalid guards gracefully halt API calls).
- **Variant Job UX Controller & Safe Editing Unlock (Slice 27)**:
  - Added `useVariantJobController` in [`use-variant-job-controller.ts`](file:///D:/Github/muhsinmuhsy/PX/PX-F/app/components/product-editor/use-variant-job-controller.ts) as the single frontend lifecycle owner for active job discovery, SSE, polling fallback, terminal callbacks, and dismissal.
  - Reduced fallback polling to 1.5 seconds for active jobs and added a 10 second idle heartbeat for active-job discovery.
  - Completed job cards auto-dismiss after 5 seconds; failed job cards persist until retry or manual dismissal.
  - Missing/orphaned variant row states now warn without freezing existing non-archived row edits.
  - Cleaned backend worker lint/type drift in [`worker.py`](file:///D:/Github/muhsinmuhsy/PX/PX-B/app/modules/catalog/worker.py) and [`test_catalog_worker.py`](file:///D:/Github/muhsinmuhsy/PX/PX-B/tests/test_catalog_worker.py), while preserving explicit SQLModel model-registry import side effects required for worker foreign-key resolution.
- **Variant Table Trust and Inventory Workspace Redesign (Slice 28)**:
  - Added [`variant-identity-cell.tsx`](file:///D:/Github/muhsinmuhsy/PX/PX-F/app/components/product-editor/variant-identity-cell.tsx) as the reusable identity renderer for ordered option chips, color/image visuals, archived state, and readable "Blue / XL" style row identity.
  - Updated the variant table to use one combined "Variant" column instead of separate option columns, so each row reads as one sellable product version.
  - Added row save confidence states (`pristine`, `dirty`, `saving`, `saved`, `failed`) through [`product-variant-workspace.ts`](file:///D:/Github/muhsinmuhsy/PX/PX-F/lib/catalog/product-variant-workspace.ts), [`admin-product-edit-form.tsx`](file:///D:/Github/muhsinmuhsy/PX/PX-F/app/components/admin-product-edit-form.tsx), and [`use-product-variant-actions.ts`](file:///D:/Github/muhsinmuhsy/PX/PX-F/app/components/product-editor/use-product-variant-actions.ts).
  - Added unsaved-change exit protection for dirty variant rows.
  - Replaced always-visible filtered/page bulk editing with selection-only bulk editing and clear selected-count copy.
  - Added i18n-backed labels for table identity, selection, save states, loading, archived restore, and bulk selection copy.
  - Removed the previous frontend ESLint warning by deleting the unused `text` parameter from the auth refresh helper in [`api.ts`](file:///D:/Github/muhsinmuhsy/PX/PX-F/lib/catalog/api.ts).
- **Exact Variant Media Binder (Slice 29)**:
  - Added [`variant-media-cell.tsx`](file:///D:/Github/muhsinmuhsy/PX/PX-F/app/components/product-editor/variant-media-cell.tsx) as the reusable table-row media picker for exact `variant_id` image binding.
  - Added an Image column to the variant table so shop managers can assign or change a variant image without leaving the inventory table.
  - Wired row media binding through the existing `PATCH /catalog/admin/products/{product_id}/media/{media_id}` API with `{ variant_id, needs_variant_rebinding: false }`.
  - Added row-level binding feedback (`saving`, `saved`, `failed`) separate from SKU/price/stock row draft state.
  - Added frontend component coverage for exact row media binding and backend API coverage rejecting cross-product variant binding.
- **Variant Refresh Intent and Draft Reconciliation (Slice 30)**:
  - Added explicit `soft` vs `hard` refresh intent to the product editor's authoritative snapshot reconciliation path.
  - Preserved dirty variant row drafts during normal soft refreshes, row saves, pagination reloads, and non-destructive job completion refreshes.
  - Added hard-refresh discard confirmation before structural/destructive actions that can invalidate row identity or replace authoritative structure.
  - Kept row saves on soft refresh so saving one variant no longer risks wiping unsaved edits in another row.
  - Simplified `useVariantPagination` into a server-row loader only; draft preservation now belongs to the editor/table state layer.
  - Added action-hook coverage proving hard-refresh actions stop before destructive APIs when the discard confirmation is declined.
- **Repair and Maintenance Panel Separation (Slice 31)**:
  - Added [`variant-maintenance-panel.tsx`](file:///D:/Github/muhsinmuhsy/PX/PX-F/app/components/product-editor/variant-maintenance-panel.tsx) as the dedicated repair/maintenance surface.
  - Replaced the everyday table's technical structure warning with a calm "Variant rows need attention" state.
  - Kept "Generate missing variants" visible as the primary safe action while moving revert, rebuild, detached-media counts, orphan counts, and removed-attribute metrics behind "Open maintenance".
  - Removed the old inline table metric helper and removed repair metrics from the empty variants state.
  - Added i18n-backed maintenance copy and frontend tests for maintenance-gated repair actions.
- **Structure Guard Payload Fix for Option Value Editing (Slice 32)**:
  - Fixed `422 Unprocessable Content` on normal option value workflows after generation/publish, including setting Blue's color and adding a new Green value.
  - Corrected frontend option and option-value create/update request types to include `expected_version` and `expected_hash`.
  - Updated create option, rename option, reorder option, add option value, and edit option value name/color/image actions to send the current structure guard.
  - Added frontend regressions proving option value create/update payloads include the guard.
  - Cleaned the generated initial Alembic migration import/type header so backend Ruff stays green; migration operations were unchanged.

### Quality Gate Compliance:
| Gate | Verification Command | Status |
| :--- | :--- | :--- |
| **Backend Tests** | `w.venv` targeted option value metadata test | **`1 passed` / `1`** for visual option value coverage |
| **Backend Linter** | `ruff check .` | **`100% clean`** (0 warnings) |
| **Backend Types** | `mypy app` | **`100% clean`** (Success: no issues in 86 source files) |
| **Frontend Tests** | `vitest run` | **`326 passed` / `326`** (100% green) |
| **Frontend Linter** | `npm run lint` | **`100% clean`** (0 warnings) |
| **Frontend Types** | `npm run typecheck` | **`100% clean`** (0 warnings) |
| **Frontend i18n** | `npm run i18n:check` | **Passed for 10 locales** |
| **Frontend Build** | `npm run build` | **`Compiled successfully`** (Production ready) |

> Backend full pytest was not re-run for Slice 32 after the previous repeated 5-minute environment timeouts. Targeted backend option-value metadata coverage passed, and backend Ruff/Mypy gates are clean.

---

## 2. Definitive Guidelines to Prevent Regressions (For New Agents)

To ensure future features (such as **Orders**, **Carts**, **Inventory**, or **Spreadsheet Importers**) do not break existing logic, every new agent/developer **MUST** strictly adhere to the following architectural rules:

### Rule 1: Never Hard-Delete Variant Rows (Soft-Delete by Default)
* **Invariant**: Standard variant removals (triggered by user clicks, option deletions, templates, or rebuilds) **must always** be soft-deleted by updating `is_archived = true` and setting `archived_at = utcnow()`.
* **Reason**: Hard-deleting rows will immediately orphan foreign keys or trigger database cascade errors on related surfaces (such as order items, analytics logs, and historical cart items).
* **When Hard Deletion is Allowed**: Only through the dedicated admin cleanup endpoint. This endpoint has a strict checklist that **must** verify:
  1. `is_archived = true`
  2. `archived_at` is older than 90 days.
  3. The variant is not referenced by media attachments, active options, or snapshot records.
  4. *Note for future Orders/Carts integration*: If you create Orders and Carts in the future, you **must immediately update** the cleanup service to verify that the variant is not linked to any order or cart before permitting hard deletion.

### Rule 2: Keep Storefront Reading Pinned to `published_version`
* **Invariant**: The public-facing storefront listing (`/catalog/storefront/...`) must **always** read variants and structures scoped to `product.published_version`. It **must never** read live draft options or active draft combinations.
* **Reason**: This guarantees transaction isolation, allowing merchants to safely draft new option combinations in the Admin Studio without their draft edits instantly leaking to and breaking their live storefront.

### Rule 3: Maintain Active-Only SKU and Combination Uniqueness
* **Invariant**: Database unique indexes are configured as **conditional indexes** (scoped to `is_archived = false`).
* **Reason**: This allows archived historical variants (which are kept for order/invoice references) to release their SKUs and option combinations. If a merchant archives `SKU-A` and then creates a new variant, they can safely assign it `SKU-A` without triggering database unique constraint violations.

### Rule 4: Do Not Allow Mutations While Jobs Are Active
* **Invariant**: All structural writes and template applications must check and reject execution if an active job (`status = queued` or `status = running`) is found for that product.
* **Reason**: Prevents race conditions and partial structural corruption.

### Rule 5: Keep Job SSE Streams Scoped to Telemetry
* **Invariant**: Do not execute database alterations or core business logic inside SSE connection handlers. The event stream is strictly a one-way telemetry system reporting the progress of decoupled worker pool tasks.

---

## 3. What is Pending (Future Hardening Slices)

The current system is ready for the next UX hardening slice. The remaining roadmap items are:
1. **Variant Editor UX Follow-up**:
   * Consider native media-to-option-value binding only if product requirements need one image to apply to every variant sharing a color/value.
2. **Richer E2E/API Test Coverage**:
   * Add Playwright or custom integration test suites confirming template Apply + Quota Failures end-to-end.
   * Add automated browser-level E2E tests for media rebinding after product rebuilds.
   * Add a unified frontend-backend integration test covering the full merchant path: *Admin Options Setup → Job Generation → Snapshot Preview → Snap Publish → Storefront Verification*.
3. **Advanced Observability**:
   * Setup production Prometheus metrics and custom alert policies for SSE reconnection rates.
   * Create dedicated server health views and job queue operational drilldowns.
4. **Stale Dead-Code Cleanup**:
   * Clean up any leftover helper files or legacy mock classes that became obsolete once the logical soft-delete archiving patterns finalized.

---

## 4. Canonical Project Paths and Modules

### Backend Core:
* [`app/modules/catalog/service.py`](file:///D:/Github/muhsinmuhsy/PX/PX-B/app/modules/catalog/service.py): Core variant, snapshot, rebuild, and logical archive/restore business logic.
* [`app/modules/catalog/models.py`](file:///D:/Github/muhsinmuhsy/PX/PX-B/app/modules/catalog/models.py): Product variant schema with conditional active-only uniqueness constraints.
* [`app/modules/catalog/repository.py`](file:///D:/Github/muhsinmuhsy/PX/PX-B/app/modules/catalog/repository.py): DB helper layers filtering storefront and catalog items.
* [`app/modules/catalog/variant_events.py`](file:///D:/Github/muhsinmuhsy/PX/PX-B/app/modules/catalog/variant_events.py): Durable Server-Sent Events bus.
* [`app/modules/catalog/worker.py`](file:///D:/Github/muhsinmuhsy/PX/PX-B/app/modules/catalog/worker.py): Asynchronous job consumer.

### Frontend Core:
* [`lib/catalog/api.ts`](file:///D:/Github/muhsinmuhsy/PX/PX-F/lib/catalog/api.ts): Storefront and Admin API client layer.
* [`app/components/admin-product-edit-form.tsx`](file:///D:/Github/muhsinmuhsy/PX/PX-F/app/components/admin-product-edit-form.tsx): Product edit state manager, tracking active jobs and handling snapshot publishing.
* [`app/components/product-editor/product-variants-section.tsx`](file:///D:/Github/muhsinmuhsy/PX/PX-F/app/components/product-editor/product-variants-section.tsx): Variants grid panel, bulk price/stock/status editing, and the dynamic `VariantJobPanel`.
* [`app/components/product-editor/variant-structure-studio.tsx`](file:///D:/Github/muhsinmuhsy/PX/PX-F/app/components/product-editor/variant-structure-studio.tsx): Interactive structure creator with rebuild previews and confirmation modals.


