# Multi-Store Support Implementation Plan

> Status: Planning Phase  
> Created: 2026-06-21  
> Owner: PX Core Team  

---

## 1. Problem Statement

The PX database schema already supports multi-tenancy at the store level (`stores` table scopes all catalog data via `store_id`). However, the entire frontend and parts of the backend assume a single store per user via hardcoded `stores[0]` access patterns.

The goal of this plan is to unlock full multi-store support in the application layer without touching the database schema or existing data.

---

## 2. Current Reality

```
User
 └── Store A (hardcoded selection)
      ├── Products
      ├── Categories
      ├── Domains
      ├── Branding
      └── Templates
```

```
User
 ├── Store A
 ├── Store B
 ├── Store C
 └── Store D
 (Database supports this — app does not)
```

### Where single-store is hardcoded

| Layer | File | Issue |
|---|---|---|
| Backend | `branding_router.py:40` | `_resolve_store_from_admin()` returns `stores[0]` |
| Frontend | `store-gate.tsx` | Redirects to create-store if 0 stores; no switch if >1 |
| Frontend | All admin pages | Each independently calls `getMyStores()` and uses `stores.stores[0]` |
| Frontend | `create-store-form.tsx` | Redirects to dashboard assuming store is now in context |
| Frontend | `(app)/layout.tsx` | No global store context provider |

No React Context exists for the selected store. Every component fetches and selects independently.

---

## 3. Proposed User Workflow

### Case 1: No Stores
```
Login
  ↓
GET /stores/me → []
  ↓
Redirect /{locale}/create-store
  ↓
Create Store
  ↓
Dashboard (Store A auto-selected)
```

### Case 2: One Store
```
Login
  ↓
GET /stores/me → [Store A]
  ↓
Store A auto-selected
  ↓
Dashboard
```

### Case 2a: Deleting the Active Store
```
User has stores: [A, B, C (active)]
  ↓
Delete Store C (confirmed)
  ↓
C is the last in ordered list → no "next"
  ↓
Select previous store (B)
  ↓
Dashboard
```

```
User has stores: [A (active), B, C]
  ↓
Delete Store A (confirmed)
  ↓
Next store exists (B)
  ↓
Select B
  ↓
Dashboard
```

Edge case:
```
User has stores: [A (active)]
  ↓
Delete Store A (confirmed)
  ↓
No stores remain
  ↓
Redirect /create-store
```

**Deletion fallback rule:**
```
if next_store exists:
    select next_store
else:
    select previous_store  (fallback when deleted store was last in ordered list)
```
This avoids an undefined state when the active store is the tail of the ordered list.

UX note:
"Select next store in ordered list" preserves user sequencing. If the user manually arranged stores as A → B → C, deleting A should land on B, not on a store that happened to be listed first. When there is no next store, fall back to the previous store rather than re-selecting the head of the list.

**Store ordering source:**
1. User-defined order (future enhancement — when store drag-sort is implemented)
2. Otherwise: `created_at` ascending (oldest first)

This ensures deterministic, predictable behavior today without requiring a new sort-field column.

**Rules:**
- If deleted store is the currently selected store:
  - If a next store exists in the ordered list → select it
  - Otherwise → select the previous store (fallback)
  - Persist new selection to localStorage
  - If no stores remain, redirect to create-store onboarding
- If deleted store is not the active store, no context switch needed

---

### Case 3: Multiple Stores (Two Sub-Options)

**Option A — Store Selection Screen:**
```
Login
  ↓
GET /stores/me → [A, B, C]
  ↓
/{locale}/stores (selection grid)
  ↓
Choose Store
  ↓
Dashboard
```

**Option B — Remember Last Selected Store (RECOMMENDED):**
```
Login
  ↓
GET /stores/me → [A, B, C]
  ↓
Read lastSelectedStoreId from localStorage
  ↓
Store B auto-selected
  ↓
Dashboard
  ↓
[Header: Store: My Electronics ▼]  ← switcher available
```

**Recommendation: Option B**  
Closest to Shopify / WooCommerce / marketplace admin UX. Keeps workflow fast, provides visible switcher.

> **Future enhancement (post-multi-store launch):**
> `localStorage` is device-specific (`laptop → Store B`, `phone → Store A`). When user accounts/profile settings are later introduced, migrate persistence to `user_preferences.selected_store_id` for cross-device consistency.

---

## 4. New UI Structure

### Header Component

```
┌─────────────────────────────────────────┐
│ Logo                                     │
│                                          │
│ Store: [ My Electronics ▼ ]             │
│                                          │
│ Account ▼                                │
└─────────────────────────────────────────┘
```

### Store Switcher Dropdown

```
┌─────────────────────┐
│ ✓ My Electronics    │   ← active store checkmark
│   Fashion Store     │
│   Furniture Store   │
│   Mobile Shop       │
│ ─────────────────── │
│ + Create New Store  │
│ Manage Stores       │
└─────────────────────┘
```

### Store Management Page

New route: `/{locale}/dashboard/stores`

```
┌─────────────────────────────────────────┐
│ My Stores                                │
│                                          │
│ ┌──────────────┐  ┌──────────────┐      │
│ │ Store A      │  │ Store B      │      │
│ │              │  │              │      │
│ │ store-a.com  │  │ store-b.com  │      │
│ │              │  │              │      │
│ │ [Switch]     │  │ [Switch]     │      │
│ │ [Edit]       │  │ [Edit]       │      │
│ │ [Domains]    │  │ [Domains]    │      │
│ │ [Delete]     │  │ [Delete]     │      │
│ └──────────────┘  └──────────────┘      │
│                                          │
│ [+ Create Store]                         │
└─────────────────────────────────────────┘
```

### Domain Management Per Store

Current:
```
One Store
One Domain
```

Proposed:
```
Store A
 ├── store-a.com       (primary)
 ├── shop.store-a.com  (verified)
 └── [+ Add Domain]

Store B
 └── store-b.com       (primary)
```

When user switches store (via header dropdown), domain page automatically shows that store's domains.

---

## 5. Store Context Architecture

### Backend

Keep the existing URL pattern for store-scoped admin routes:

```
/admin/stores/{store_id}/products
/admin/stores/{store_id}/categories
/admin/stores/{store_id}/domains
```

The backend already understands `store_id` from the path. The frontend Store Context simply provides `selectedStore.id`, and all admin API calls embed it in the URL. This avoids introducing a new middleware/header contract across the entire admin API.

For public storefront:
- Continue resolving store from `Host` header (`slug.base_domain`) — this is correct and unchanged

**Backend files to change:**

| File | Change |
|---|---|
| `branding_router.py` | Replace `stores[0]` with store resolved from `x-store-id` header (only exception to URL-pattern rule; see note below) |
| `stores/router.py` | None required (already returns all stores for user) |

**Note on `branding_router.py`:**
The branding router uses a flat prefix (`/catalog/admin/branding`) without a `{store_id}` path param. The minimal-change fix is to have the frontend send `x-store-id: <uuid>` on these calls, and add a small helper to extract/validate it. This avoids restructuring the branding URL hierarchy.

**Branding Subsection (Detailed)**

Unlike all other admin modules, branding cannot use the `/admin/stores/{store_id}/...` path pattern because its endpoints are structured as:

```
GET    /catalog/admin/branding
PATCH  /catalog/admin/branding
POST   /catalog/admin/branding/media/upload-url
POST   /catalog/admin/branding/media
GET    /catalog/admin/branding/media
```

**Frontend change — `PX-F/lib/stores/api.ts`:**

All 5 branding functions (`getBranding`, `saveBranding`, `createBrandMediaUploadUrl`, `createBrandMedia`, `listBrandMedia`) must include the `x-store-id` header. The `request()` helper in this project already supports credential mode; confirm it allows custom headers and add `x-store-id: selectedStore.id` to each call.

**Backend change — `PX-B/app/modules/stores/branding_router.py`:**

1. Add a FastAPI dependency (or simple header read) that extracts `x-store-id` from the request
2. Validate that the store belongs to the current user (compare against `list_stores_by_owner`)
3. Pass the resolved `Store` into each endpoint instead of calling `_resolve_store_from_admin()`
4. Remove or deprecate `_resolve_store_from_admin()` — it is no longer needed

**Endpoints affected by branding backend change:**
- `GET /catalog/admin/branding` → `get_branding`
- `PATCH /catalog/admin/branding` → `update_branding`
- `POST /catalog/admin/branding/media/upload-url` → `create_brand_media_upload_url`
- `POST /catalog/admin/branding/media` → `create_brand_media`
- `GET /catalog/admin/branding/media` → `list_brand_media`

**Branding tests to update:**
- `PX-F/lib/stores/__tests__/api.test.ts` — assert `x-store-id` header sent
- Backend tests for `branding_router.py` — mock header and assert correct store is used

---

**Other multi-store-aware but incomplete APIs:**
`get_variant_job_metrics_admin` in `catalog/router.py:1636` accepts an **optional** `store_id` query parameter. When omitted, it falls back to `list_stores_by_owner(session, current_user.id)` and aggregates metrics across all owned stores. This is safe today for single-store users but would produce cross-store-aggregated numbers once a user has multiple stores. The migration must ensure `variant-job-metrics-panel.tsx` always passes an explicit `storeId`.

### Frontend

Create a **Store Context** at the app root level.

**New files:**

| File | Purpose |
|---|---|
| `PX-F/app/contexts/store-context.tsx` | React Context + Provider for selected store |
| `PX-F/app/hooks/use-selected-store.ts` | Consumer hook for selected store |
| `PX-F/app/components/header/store-switcher.tsx` | Dropdown in admin header |
| `PX-F/app/(app)/layout.tsx` | Wrap in StoreProvider, render StoreSwitcher |

**Provider responsibility:**
- On mount, call `GET /stores/me`
- Load `lastSelectedStoreId` from `localStorage`
- If match found in list, select it; otherwise select first store
- On store switch: update context, persist `lastSelectedStoreId` to `localStorage`, invalidate all store-scoped query caches
- Expose: `selectedStore`, `stores`, `switchStore(storeId)`, `isLoading`
- **Audit rule**: `selectedStore.id` must be passed explicitly to every admin API call. No call should rely on a backend no-`storeId` default.

**Change all admin pages from:**
```tsx
const { stores } = useMyStores();
const storeId = stores.stores[0].id;
```
**To:**
```tsx
const { selectedStore } = useSelectedStore();
const storeId = selectedStore.id;
```

---

## 6. Store Management Page

### Route
```
/{locale}/dashboard/stores           ← list + create
/{locale}/dashboard/stores/[id]     ← edit detail
```

### Features
- Grid/list of owned stores with name, slug, domain(s), template
- Switch button → calls `switchStore(storeId)`, navigates to dashboard
- Edit button → navigate to store detail page
- Create Store button → opens modal or navigates to create page
- Delete store (with confirmation; prevent deleting last store)
- Per-store domain management (links to existing domains page with store context)

---

## 7. Pricing (Future — No Implementation Now)

The database already has `tier` on `stores`. Future plans:

| Plan | Store Limit |
|---|---|
| Free | 1 |
| Starter | 3 |
| Business | 10 |
| Enterprise | Unlimited |

For now: **no store limits**. The account holds unlimited stores.

---

## 8. Implementation Scope

### Backend Changes

1. **`branding_router.py`** — Replace `_resolve_store_from_admin()` `stores[0]` with a FastAPI dependency that reads `x-store-id` header directly from the request, validates it belongs to the current user, and returns the Store. No global middleware required.
2. **`stores/router.py`** — None required (already returns all stores for user)

### Frontend Changes

1. **`store-context.tsx`** — New context provider
2. **`use-selected-store.ts`** — New hook
3. **`(app)/layout.tsx`** — Wrap children in StoreProvider, add StoreSwitcher to header
4. **`store-gate.tsx`** — Update gate logic for multi-store case (redirect to selection if needed)
5. **`create-store-form.tsx`** — After creation, switch context to new store instead of redirecting blindly
6. **All admin pages** — Replace `getMyStores()` / `stores[0]` with `useSelectedStore()`
   - `custom-domains-panel.tsx`
   - `current-store-domain-card.tsx`
   - `store-locales-panel.tsx`
   - `store-templates-panel.tsx`
   - `brand-settings-page.tsx`
   - `admin-product-edit-form.tsx`
   - `admin-product-create-form.tsx`
   - `admin-products-list.tsx`
   - `admin-category-edit-form.tsx`
   - `admin-category-create-form.tsx`
   - `admin-categories-list.tsx`
 7. **`variant-job-metrics-panel.tsx`** — Pass explicit `storeId` to `getVariantJobMetrics` (currently calls without storeId, would aggregate across all stores)
 8. **`navbar.tsx` or header component** — Add StoreSwitcher component
 9. **New page: `/{locale}/dashboard/stores`** — Store management list

---

## 9. Store Switch Handling

When the user switches stores, stale data from the previous store must be purged to avoid leaking Store A data into Store B views.

### Actions on `switchStore(storeId)`

1. **Invalidate query caches**
   - React Query / SWR: remove all cached responses keyed by the previous `storeId`
   - Clear product lists, category lists, branding, domains, metrics, and variant jobs

2. **Clear unsaved form state**
   - Draft product forms
   - Category forms
   - Variant matrix drafts
   - Any local component state not yet submitted

3. **Refetch critical data**
   - Dashboard metrics
   - Products list
   - Categories list
   - Branding settings
   - Domains list
   - Locales / template settings

### Frontend integration

The `switchStore` function in `store-context.tsx` should:

```ts
const switchStore = (storeId: string) => {
  if (selectedStore?.id === storeId) return;
  setSelectedStore(stores.find(s => s.id === storeId));
  localStorage.setItem(KEY, storeId);
  queryClient.removeQueries({
    predicate: (query) => query.queryKey[0] === 'stores' && query.queryKey[1] !== storeId,
  });
  // Optionally trigger immediate refetch for visible pages
};
```

---

## 10. Implementation Phases

### Phase 1 — Frontend Foundation (Unlocks multi-store UX)
1. **Store Context + Switcher UI** — React Context + hook + header dropdown
2. **API Audit** — Search entire frontend and backend for multi-store risks (see checklist below)
3. **Migrate all admin pages** — Replace `stores[0]` with `useSelectedStore()`. Ensure all `storeId`-bearing API calls pass `selectedStore.id` explicitly.

#### Multi-Store Hardcoding Checklist (Phase 1 Audit)

Search and confirm no other hidden single-store assumptions exist beyond what is already documented.

| Pattern | Status |
|---|---|
| `stores[0]` in frontend components | ⚠️ Known — 8 admin components |
| `stores.stores[0]` in frontend tests | ⚠️ Known — 6 test files |
| `list_stores_by_owner(...)[0]` in backend | ⚠️ Known — `branding_router.py:40` |
| `stores[0]` in backend service logic | ❓ Audit required |
| SQLAlchemy `.first()` on store queries | ❓ Audit required |
| SQL `LIMIT 1` on store queries | ❓ Audit required |
| `scalar_one_or_none()` on store queries | ❓ Audit required |
| APIs with implicit no-storeId fallback | ⚠️ Known — `getVariantJobMetrics` |
| Any other `[0]` selection on store list | ❓ Audit required |

After audit, document all findings. Any new hits must be added to Phase 1 migration list or Phase 3 backend fixes.

> Once this phase is done, most multi-store functionality is already unlocked.

### Phase 2 — Store Management (User visibility & control)
3. **Store Management page** — List/create/edit/delete stores
4. **Store Create Flow** — Auto-select new store on creation
5. **Store Delete Flow** — Handle active-store deletion:
    - Deleted store == selected → if next store exists, select next; else select previous
    - No stores remain → redirect to create-store

### Phase 3 — Backend & Risk Fixes
6. **Branding router fix** — Remove `stores[0]` assumption; accept `x-store-id` header
7. **API Audit** — Search for all endpoints with implicit no-storeId defaults (e.g. `getVariantJobMetrics`)
8. **VariantJobMetricsPanel** — Make `storeId` required, pass `selectedStore.id`

### Phase 4 — Polish & Testing
9. **Cache invalidation** — Store switch handling (clear React Query/SWR, refetch)
10. **Tests** — Multi-store admin flows, deletion edge cases, cache clearing

---

## 11. Success Criteria

- A user can create multiple stores
- User can switch between stores via header dropdown
- All admin pages (products, categories, branding, domains, templates, locales) work correctly on the selected store
- Store context persists across page navigations and browser refreshes
- Creating a new store automatically selects it
- Store management page lists all stores with switch/edit/delete actions
- Deleting the active store auto-selects another remaining store (or triggers create-store flow if none remain)
- Backend `branding_router.py` no longer uses `stores[0]`
- `getVariantJobMetrics` always called with explicit `storeId` — no cross-store aggregation
- All admin API requests become store-explicit: no endpoint relies on implicit "first store" fallback behavior
- No regression in single-store flow for existing users
- All existing tests continue to pass

---

## 12. Open Questions (Resolved)

| Question | Decision |
|---|---|
| Should store switching invalidate any cached data in the frontend? | **Yes** — store-specific data should always be cleared and refetched |
| Should there be a warning or confirmation when switching away from an unsaved form? | **Yes** — show modal: "You have unsaved changes. Leave this store anyway?" [Stay] [Switch Store] |
| Should the store management page be a modal or full page? | **Full page** — Stores, Domains, Templates, Branding, Analytics will grow over time; full page scales better than a modal |

---

## 13. Out of Scope

- Store-level pricing / plan enforcement (deferred)
- Per-store user roles and permissions beyond ownership
- Multi-store analytics aggregation
- Store-level checkout settings (future)


## 14. IMP

The code must be well-maintained, reusable, organized, and structured to a production-level standard. We already have an existing architecture and workflow—please adhere to them and maintain consistency across the system.


Also, since the system supports multiple languages, ensure proper internationalization is followed throughout.


In addition:


Add comprehensive test cases at all levels (both frontend and backend).
Remove any unused or dead code.
Ensure the codebase is clean and maintainable.
Make sure everything passes: build, tests, type checks, and linting for both frontend and backend.
