# Brand Settings Plan

Status: Proposed
Created: 2026-06-07
Scope: Store branding configuration (logo, favicon) in admin, reflected on customer-facing storefront. Admin panel branding remains owned by the platform.

## Purpose

Establish a Brand Settings surface in the admin so merchants can customize their storefront identity without affecting the admin panel experience. The platform retains full control over the admin dashboard branding.

## System Context

The PX system already separates two UI surfaces:

- **Admin** (`/admin/...`): The platform-operated merchant dashboard. Currently uses platform branding and is not customizable by tenants.
- **Storefront** (`/storefront/{store_domain}/...`): The customer-facing shop. Currently lacks a structured branding configuration surface.

Variant, product, and store domain work is complete through Slice 33. The database already has store scoping, tenant isolation, and media attachment infrastructure suitable for a brand settings feature.

## Requirements

### Primary

- Add a Brand Settings entry to the admin side menu.
- Allow merchants to upload and persist a store logo.
- Allow merchants to upload and persist a favicon.
- Store branding must render on the customer-facing storefront pages.
- Admin panel branding must remain fixed as the platform brand (not tenant-customizable).

### Design Decision: Separate Media Tables

Brand media and product media are intentionally separated into distinct database tables.

```
store_branding_media   ← new, dedicated to store-level branding assets
product_media          ← existing, for product and variant images only
```

**Rationale:**

Brand assets (logo, favicon, banners, hero images, marketing assets, email branding, social sharing images, theme assets) are store-level concerns, not product-level concerns. Mixing them into `product_media` works technically but creates conceptual debt: a logo row would sit in the product media table with `product_id = null` and `variant_id = null`, forcing future developers to interpret "null product_id = maybe a logo" as a special-case rule. As the platform grows beyond logo and favicon (store banners, hero images, marketing assets), the separation becomes increasingly important.

A dedicated table provides:
- Clear ownership and query intent — every query is explicit about what it touches.
- No accidental mixing — brand assets can never appear in product media listings, and product images can never be selected as store logos.
- Simpler permissions and audit trails at the DB layer.
- Natural expansion path for future brand asset types without polluting product media.

**Out of Scope (MVP)**

The following branding capabilities are intentionally excluded from this slice and may be added in future iterations:

- Brand colors
- Footer customization
- Social links
- Typography
- Custom CSS
- Email branding
- Multi-brand support

### Non-Negotiables

- Admin and storefront branding are independent. Changing store branding must not alter admin appearance.
- Brand assets are store-scoped and respect tenant isolation.
- Brand media is stored in a dedicated `store_branding_media` table, separate from `product_media`. No brand asset lives inside the product media namespace.
- Uploaded assets go through the existing media/service layer; do not introduce a new file-serving path.
- Brand settings should not break existing storefront rendering when assets are missing.

## Proposed UX

- New admin page entry under Settings: **Brand Settings**.
- Prominent notice at top of page:

  > These settings affect your customer-facing storefront only. They do not change the appearance of the admin dashboard.

- **Form fields:**
  - Store logo upload with preview.
  - Favicon upload with preview.
- **Action:** Save branding button with optimistic feedback and error handling.
- **Empty state:** show current asset placeholders or neutral "Not set" state.

## Backend Proposal

### Database

**New tables: `store_branding_settings` (1:1 with Store) and `store_branding_media` (1:many with Store).**

```
store_branding_settings
 ├─ store_id: UUID, PK, FK → stores.id
 ├─ logo_media_id: UUID | None, FK → store_branding_media.id
 ├─ favicon_media_id: UUID | None, FK → store_branding_media.id
 ├─ created_at: datetime
 └─ updated_at: datetime

store_branding_media
 ├─ id: UUID, PK
 ├─ store_id: UUID, FK → stores.id (indexed)
 ├─ media_type: str (e.g. "logo", "favicon")
 ├─ storage_provider: str (default "external")
 ├─ storage_key: str (max 512 chars)
 ├─ public_url: str (max 2048 chars)
 ├─ mime_type: str | None
 ├─ size_bytes: int | None
 ├─ width: int | None
 ├─ height: int | None
 ├─ alt_text: str | None
 ├─ sort_order: int
 ├─ is_active: bool (default True)
 ├─ archived_at: datetime | None
 ├─ created_at: datetime
 └─ updated_at: datetime
```

Optional future constraint: a unique index on `(store_id, media_type)` where `is_active = True` enforces one active asset per type per store. Omit this constraint if versioning or multiple simultaneous assets per type is planned — the `media_type` field already supports that without schema changes.

Do not store raw image bytes. `store_branding_media` mirrors the storage and serving fields of `product_media` but is isolated to store-level assets only — it has no `product_id`, `variant_id`, `is_primary`, `needs_variant_rebinding`, or any product-related fields.

`store_branding_settings` holds the FK references (`logo_media_id`, `favicon_media_id`) to `store_branding_media` rows. This keeps the branding settings shape independent of the media storage shape, allowing multiple brand assets per store in the future (banners, hero images, etc.) without schema changes to the settings table.

**Lifecycle:**

- One `store_branding_settings` record per store.
- Created lazily on first `PATCH`.
- `GET` returns default/null branding values when no record exists yet.
- `store_branding_media` rows are created per upload and referenced by `logo_media_id` / `favicon_media_id`. When a branding asset is replaced, the previous `store_branding_media` row is soft-archived (`is_active = False`, `archived_at` set). Hard deletion of unreferenced rows is not performed automatically; a separate cleanup job or cascade rule should be defined in implementation if storage cost is a concern.

**Why not reuse `product_media`:**

`product_media` is structurally bound to the product/variant domain (`product_id`, `variant_id`, `is_primary`, `needs_variant_rebinding`). Brand assets have no product association and would require null sentinel values in product-specific columns — a design smell that worsens as more brand asset types are added. Separate tables preserve domain clarity and prevent accidental leakage of brand assets into product media queries.

### API

Backend is the source of truth. The store is resolved from the authenticated admin context; **do not** expose `store_id` in the URL for admin branding routes.

Add admin endpoints under the authenticated store scope:

- `GET /catalog/admin/branding` — returns `store_branding_settings` record
- `PATCH /catalog/admin/branding` — updates `logo_media_id` / `favicon_media_id`
- `POST /catalog/admin/branding/media/upload-url` — generates upload URL for brand media (analogous to the existing product media upload endpoint)

Request/response should include only identifiers and metadata (not media blobs). Media reads should use the existing media serving path.

**Example `PATCH` request body:**

```json
{"logo_media_id": "uuid-or-null", "favicon_media_id": "uuid-or-null"}
```

`null` values are valid and should remove the current asset reference, causing storefront fallback to defaults.

**Example `GET` response body:**

```json
{
  "logo_media_id": "uuid-or-null",
  "favicon_media_id": "uuid-or-null",
  "created_at": "2026-06-07T10:00:00Z",
  "updated_at": "2026-06-07T10:00:00Z"
}
```

**Upload flow:**

1. Client calls `POST /catalog/admin/branding/media/upload-url` with `filename`, `mime_type`, `media_type` (e.g. `"logo"` or `"favicon"`). The store is resolved from the authenticated admin context — **no `store_id` in the request body or URL**.
2. Backend resolves the merchant's store and generates a `storage_key` under `stores/{store_id}/branding/{media_type}/{random_suffix}-{slugified_filename}` and returns a signed upload URL.
3. Client PUTs bytes to the upload URL.
4. Backend creates a `store_branding_media` row scoped to the resolved store.
5. Client calls `PATCH /catalog/admin/branding` with the new `media_id`.

This is structurally parallel to the existing product media upload flow but routes through a dedicated branding path to keep the namespaces separate.

### Validation & Guardrails

- Enforce store ownership on brand read/write.
- Branding updates are independent of variant generation, publishing, and background processing workflows and should not be blocked by them.
- Validate `media_id` references belong to the same store before saving.
- **Validate media type consistency:** `logo_media_id` must reference a `store_branding_media` row with `media_type = "logo"`, and `favicon_media_id` must reference a row with `media_type = "favicon"`. Reject mismatched assignments. This prevents a favicon image from being linked as the store logo or vice versa.
- **Validate asset formats:**
  - Logo: PNG, JPG, SVG
  - Favicon: ICO, PNG, SVG (subject to storefront stack support; recommended size: 32x32 or 48x48)
- SVG uploads must follow the existing media security and sanitization pipeline to prevent script injection or unsafe markup.
- **Graceful fallback:** if a referenced media asset is deleted, archived, or inaccessible, the branding setting should fall back to the default storefront placeholder state instead of rendering a broken image or favicon.

## Frontend Proposal

### Navigation

- Add Brand Settings to the admin settings side menu.
- Keep the existing admin sidebar behavior and locale/i18n patterns.

### Admin Page

- New settings form component under the admin routes.
- Use existing media picker/uploader patterns where possible.
- Show preview of current logo and favicon.
- Save updates via the new branding API.
- Show localized success/error states consistent with existing admin UX.

### Storefront Rendering

- Update storefront shell/header and HTML `<head>` to read branding from store data.
- Use a neutral fallback when no logo/favicon is set (current placeholder behavior).

**Rendering rules:**

- Store logo: displayed in the storefront header.
- Favicon: rendered as the HTML head favicon link.
- Keep storefront reads within the existing storefront scoping rules (`published_version`, store domain routing).

## Files Likely To Touch

> This is a scoping list, not a final diff. Verify against live code before implementation.

**Backend (PX-B):**

- `app/modules/stores/models.py` — new `StoreBrandingSettings` and `StoreBrandingMedia` models
- `app/modules/stores/schemas.py` — branding request/response schemas
- `app/modules/stores/router.py` — `GET /catalog/admin/branding`, `PATCH /catalog/admin/branding`, `POST /catalog/admin/branding/media/upload-url`
- `app/modules/catalog/service.py` — extend or mirror the existing `build_media_upload_url()` for brand media storage key generation (`stores/{store_id}/branding/...`)
- Alembic migration for `store_branding_settings` and `store_branding_media` tables
- `app/modules/stores/service.py` — store branding business logic (ownership checks, fallback logic)

**Frontend (PX-F):**

- Admin settings navigation and route
- `lib/stores/api.ts` — branding endpoints and brand media upload
- `lib/stores/types.ts` — branding and brand media types
- `components/admin/brand-settings/` — Brand Settings page, upload form, logo/favicon preview components
- Storefront shell/head components for logo/favicon rendering (reads from store data, resolves via `storeBrandingMedia`)
- `lib/i18n/ui-copy.ts` and locale dictionaries for new copy
- Tests in `app/components/.../__tests__/` and `lib/stores/__tests__/`

Note: The existing `productMedia` API, types, and components are **not** modified. Brand media uses its own API surface (`/catalog/admin/branding/...`) and own React data structures.

## Migration Plan

- Two new tables: `store_branding_settings` (1:1 with Store) and `store_branding_media` (1:many with Store, one row per brand asset).
- Both tables use safe default `NULL` values; no data backfill required — merchants set branding explicitly.
- New upload endpoint routes brand media to `stores/{store_id}/branding/...` storage keys, physically separated from `stores/{store_id}/products/...` on disk or in bucket.
- **Downtime risk:** low. Feature is additive and bypasses variant/publish paths. Existing `product_media` table is untouched.

## Future Extensions

- Store banner, homepage hero image — natural additions to `store_branding_media`.
- Marketing assets (campaign banners, promotional graphics).
- Email branding assets (header image, logo).
- Social sharing images (Open Graph / Twitter Card images).
- Theme assets (custom CSS, fonts) — structured in brand settings, separate from product data.
- Multi-brand/white-label for platform-managed store clusters.
- Preview pane showing live storefront snippet before save.
- Brand asset versioning or scheduled rollover.

All of these fit naturally into `store_branding_media` without touching `product_media`.

## Risks & Considerations

- Media FK safety: ensure cleanup respects archive rules.
- Cache invalidation: a successful branding update **must** trigger storefront cache invalidation. Branding changes should be visible on the storefront immediately or within the platform's normal cache refresh window. The backend is responsible for issuing the invalidation signal after `PATCH /catalog/admin/branding` commits.
- i18n coverage for all supported locales via the existing `npm run i18n:check` gate.
- Keep admin vs storefront separation clear to avoid tenant expectations that the admin panel can be rebranded.

## Acceptance Criteria

- Merchants can set store logo and favicon from admin Brand Settings.
- Storefront renders merchant branding; admin remains platform-branded.
- Brand assets are store-scoped and tenant-isolated.
- Missing assets render neutral fallbacks without errors.
- Feature is additive and does not change existing variant, product, or media behavior.
- Branding updates are visible on the storefront after cache refresh/invalidation.

## Validation

### Backend Commands

```bash
cd PX-B
python -m dotenv -f .env.local.dev run -- pytest -v
ruff check .
mypy app
```

### Frontend Commands

```bash
cd PX-F
npm test
npm run lint
npm run typecheck
npm run i18n:check
npm run build
```

## Testing Strategy

### Backend Tests

- Branding CRUD under authenticated store context.
- Store ownership enforcement; cross-tenant access rejected.
- Lazy record creation on first `PATCH`.
- `GET` returns default/null shape when no record exists.
- Null `logo_media_id` / `favicon_media_id` removes asset reference.
- Invalid `media_id` references rejected.
- Archived or deleted media reference falls back to placeholder behavior.
- Slug/domain routing remains unchanged and storefront continues to resolve via `published_version` rules.

### Frontend Tests

- Brand Settings page renders notice, form fields, previews, and save feedback.
- Logo/favicon preview updates after successful save.
- Save failure shows localized error state.
- Empty branding state renders neutral placeholders.
- Side menu entry present and locale/i18n-clean.
- `npm run i18n:check` passes for all supported locales.

### Storefront Rendering Tests

- Store logo renders in storefront header when set.
- Favicon renders in HTML `<head>` when set.
- Missing logo/favicon renders neutral placeholder without errors.
- Archived media asset causes storefront fallback, not broken image/favicon.
- Storefront branding remains independent from admin panel appearance.

### Regression Tests

- Merchant removes logo -> storefront falls back to default correctly.
- Merchant removes favicon -> storefront falls back to default correctly.
- `PATCH` with `logo_media_id` belonging to another store is rejected.
- `PATCH` with `favicon_media_id` belonging to another store is rejected.
- Archived or deleted media asset reference -> storefront falls back to default instead of rendering a broken asset.
