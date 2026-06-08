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

### Out of Scope (MVP)

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

Add a lightweight store branding record or extend the existing store model. Preferred approach:

**New table `store_branding_settings` scoped 1:1 with Store.**

Fields:

- `store_id`: UUID, PK, FK to `stores.id`
- `logo_media_id`: UUID | None, FK to `media.id`
- `favicon_media_id`: UUID | None, FK to `media.id`
- `created_at`: datetime
- `updated_at`: datetime

Do not store raw image bytes; reference existing media attachments through FK to remain consistent with the current media architecture.

**Lifecycle:**

- One branding record per store.
- Created lazily on first `PATCH`.
- `GET` returns default/null branding values when no record exists yet.

> Alternatively, if the store model is already the natural home and no brand-specific query pattern demands its own table, store branding fields can be added directly to `Store`. Use a dedicated table if branding settings are expected to grow (colors, fonts, etc.).

### API

Backend is the source of truth. The store is resolved from the authenticated admin context; **do not** expose `store_id` in the URL for admin branding routes.

Add admin endpoints under the authenticated store scope:

- `GET /catalog/admin/branding`
- `PATCH /catalog/admin/branding`

Request/response should include only identifiers and metadata (not media blobs). Media reads should use the existing media API.

**Example `PATCH` request body:**

```json
{"logo_media_id": "uuid-or-null", "favicon_media_id": "uuid-or-null"}
```

`null` values are valid and should remove the current asset, causing storefront fallback to defaults.

**Example `GET` response body:**

```json
{
  "logo_media_id": "uuid-or-null",
  "favicon_media_id": "uuid-or-null",
  "created_at": "2026-06-07T10:00:00Z",
  "updated_at": "2026-06-07T10:00:00Z"
}
```

### Validation & Guardrails

- Enforce store ownership on brand read/write.
- Branding updates are independent of variant generation, publishing, and background processing workflows and should not be blocked by them.
- Validate `media_id` references belong to the same store before saving.
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

- `app/modules/catalog/models.py` or a new `app/modules/stores/models.py/stores_branding.py`
- Alembic migration for new table or columns
- `app/modules/catalog/schemas.py` or `app/modules/stores/schemas.py`
- `app/modules/catalog/router.py` or new `app/modules/stores/router.py`
- Store service layer if brand rules need business logic

**Frontend (PX-F):**

- Admin settings navigation and route
- `lib/catalog/api.ts` for branding endpoints
- `lib/catalog/types.ts` for branding types
- Storefront shell/head components for logo/favicon rendering
- `lib/i18n/ui-copy.ts` and locale dictionaries for new copy
- Tests in `app/components/.../__tests__/` and `lib/catalog/__tests__/`

## Migration Plan

- New table migration with safe default `NULL` values for both media IDs.
- No data backfill required; merchants set branding explicitly.
- **Downtime risk:** low. Feature is additive and bypasses variant/publish paths.

## Future Extensions

- Primary/accent colors, typography, custom CSS.
- Multi-brand/white-label for platform-managed store clusters.
- Preview pane showing live storefront snippet before save.
- Brand asset versioning or scheduled rollover.

## Risks & Considerations

- Media FK safety: ensure cleanup respects archive rules.
- Cache invalidation for storefront headers after brand update. Branding updates should become visible on storefront immediately or within the platform's normal cache refresh window.
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
