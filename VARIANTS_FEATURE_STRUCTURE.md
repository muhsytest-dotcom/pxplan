# Variants Feature Codebase Structure

Complete mapping of the variants feature across PX-B (backend) and PX-F (frontend) applications.

---

## Backend Structure (PX-B)

### 1. Database Models
**Location:** [PX-B/app/modules/catalog/models.py](PX-B/app/modules/catalog/models.py)

#### Core Variant Models:
- **`CatalogVariantJobType`** (StrEnum) - Lines 33-36
  - `GENERATE_MISSING_VARIANTS`: Auto-generate variants from option selections
  - `REBUILD_VARIANTS`: Rebuild entire variant set, handles cleanup

- **`CatalogVariantJobStatus`** (StrEnum) - Lines 38-45
  - `QUEUED`, `RUNNING`, `COMPLETED`, `FAILED`, `CANCELLED`

- **`CatalogVariantJobPhase`** (StrEnum) - Lines 46-62
  - Execution phases: QUEUED → VALIDATING → COMPUTING_COMBINATIONS → CREATING_VARIANTS → DETACHING_MEDIA → DELETING_VARIANT_ATTRIBUTES → REMOVING_VARIANTS → FINALIZING → COMPLETED/FAILED/CANCELLED

- **`CatalogVariantJob`** (SQLModel) - Lines 225-253
  - Table: `catalog_jobs`
  - Tracks async variant generation jobs with status, phases, counts (created/removed/preserved)
  - Stores input snapshot for reproducibility
  - Fields: `id`, `store_id`, `product_id`, `created_by_user_id`, `type`, `status`, `phase`, created/removed/preserved counts, media rebind count, error messages, timestamps

- **`VariantTemplate`** (SQLModel) - Lines 308-336
  - Table: `variant_templates`
  - System or store-scoped templates defining variant option combinations
  - Supports system templates (is_system=true, store_id=NULL) and custom store templates
  - Fields: `id`, `store_id`, `is_system`, `system_key`, `group_name`, `name`, `description`

- **`VariantTemplateOption`** (SQLModel) - Lines 337-363
  - Table: `variant_template_options` (junction table)
  - Links template to option library options with position ordering
  - Unique constraint: (template_id, option_id)

- **`ProductVariant`** (SQLModel) - Lines 423-448
  - Table: `product_variants`
  - Individual variant combination with SKU, pricing override, stock quantity
  - `combination_key`: Sorted unique identifier for option value combinations (e.g., "val1|val2|val3")
  - `is_active`: Boolean flag for active variants
  - Unique constraints: (product_id, sku) and (product_id, combination_key)

- **`VariantValueLink`** (SQLModel) - Lines 449-463
  - Table: `variant_value_links` (junction table)
  - Links ProductVariant to ProductOptionValue selections
  - Unique constraint: (variant_id, option_value_id)

#### Related Models (also in models.py):
- **`OptionLibraryOption`** - Lines 265-305: System/store-scoped options (Size, Color, etc.)
- **`OptionLibraryValue`** - Lines 264-288: Values for option library options (S/M/L for Size, #FF0000 for Color, etc.)
- **`ProductOption`** (not shown in excerpt): Product-specific options inheriting from option library
- **`ProductOptionValue`** (not shown in excerpt): Product-specific values for options

### 2. Database Schema
**Location:** [PX-B/app/modules/catalog/schemas.py](PX-B/app/modules/catalog/schemas.py)

#### Request/Response Schemas:

**Variant Creation & Updates:**
- `ProductVariantCreateRequest` (Line 455) - Fields: `combination_key`, `sku`, `price_override`, `stock_qty`, `weight`, `is_active`
- `ProductVariantUpdateRequest` (Line 481) - Partial update fields
- `ProductVariantRead` (Line 517) - Full variant response with selections

**Variant Listing:**
- `ProductVariantListResponse` (Line 532) - Paginated list of ProductVariantRead

**Variant State:**
- `ProductVariantOptionSelectionRead` (Line 507) - Individual option value selection within variant
- `VariantOccupancyProjection` (mentioned) - Tracks existing variant combinations

**Template Management:**
- `VariantTemplateRead` (Line 271) - Template data with options
- `VariantTemplateListResponse` (Line 286) - List of templates
- `VariantTemplateApplyRequest` (Line 295) - Request to apply template to product
- `VariantTemplateOptionRead` (Line 259) - Option within template

**Rebuild Operations:**
- `ProductVariantRebuildResponse` (Line 539) - Response after rebuild job completion
- `ProductVariantRebuildImpactRead` (Line 546) - Preview of rebuild impact (created/removed/preserved counts)

**Job Management:**
- `CatalogVariantJobRead` (Line 557) - Job status, progress, phase information
- `ProductVariantStructureRevertPreviewRead` (Line 588) - Preview of revertable additions
- `ProductVariantStructureRevertResponse` (Line 600) - Response from revert operation
- `VariantDeleteImpactRead` (Line 606) - Impact analysis of deleting a variant value

### 3. Repository Layer (Data Access)
**Location:** [PX-B/app/modules/catalog/repository.py](PX-B/app/modules/catalog/repository.py)

#### Job Management Functions:
- `create_catalog_variant_job()` (Line 53) - Create new job record
- `save_catalog_variant_job()` (Line 63) - Persist job updates
- `get_catalog_variant_job_by_id()` (Line 73) - Retrieve job by ID
- `get_active_catalog_variant_job_for_product()` (Line 79) - Get currently active job for product

#### Template Functions:
- `create_variant_template()` (Line 482) - Create template
- `get_variant_template_by_id()` (Line 492) - Retrieve template
- `list_variant_templates()` (Line 498) - List templates with filtering (system/store)
- `get_variant_template_by_system_key()` (Line 522) - Get system template by key
- `create_variant_template_option()` (Line 532) - Add option to template
- `list_variant_template_options()` (Line 542) - List options in template

#### Variant Functions:
- `create_product_variant()` (Line 721) - Create variant row
- `get_product_variant_by_id()` (Line 731) - Retrieve variant
- `list_product_variants()` (Line 737) - List variants for product
- `count_product_variants()` (Line 767) - Count variants for product
- `delete_product_variant()` (Line 774) - Delete variant
- `list_variant_occupancy_projections()` (Line 756) - Get existing combination keys

#### Value Link Functions:
- `create_variant_value_link()` (Line 782) - Link variant to option values
- `list_variant_value_links()` (Line 792) - Get all value links for variant
- `delete_variant_value_link()` (Line 803) - Remove link

### 4. Service Layer (Business Logic)
**Location:** [PX-B/app/modules/catalog/service.py](PX-B/app/modules/catalog/service.py)

#### Core Variant Operations:
- `create_product_variant_row()` (Line 1534) - Create and validate variant record
- `generate_missing_variants_for_product()` (Line 1819) - Auto-generate variants from option structure
- `rebuild_variants_for_product()` (Line 1948) - Rebuild entire variant set, handling orphaned variants
- `update_variant_for_product()` (Line 2299) - Update variant fields (price, stock, etc.)
- `delete_product_variant_row()` (Line 2337) - Delete variant with impact analysis
- `list_variants_for_product()` (Line 1886) - Retrieve variants with pagination

#### Variant Template Operations:
- `list_variant_templates_with_options()` (Line 1158) - Get templates with full option details
- `apply_variant_template_to_product()` (Line 1196) - Apply template to product (creates options/values)
- `preview_apply_variant_template_impact()` (Line 1355) - Preview template application impact

#### Async Job Execution:
- `create_catalog_variant_job_for_product()` (Line 1599) - Create job record with input snapshot
- `get_catalog_variant_job_for_admin()` (Line 1647) - Retrieve job for admin
- `get_active_catalog_variant_job_for_admin_product()` (Line 1653) - Get active job
- `execute_catalog_variant_job()` (Line 1659) - **Main job executor** - Handles full job lifecycle:
  - Validates structure guard (prevents stale operations)
  - Computes combinations
  - Detaches media if rebuilding
  - Deletes variant attributes if rebuilding
  - Removes orphaned variants if rebuilding
  - Generates/rebuilds variants
  - Tracks counts and errors
  - Updates job status through phases

- `_mark_catalog_variant_job()` (Line 1773) - Update job status, phase, and metrics

#### Impact Analysis:
- `preview_rebuild_variants_impact()` (Line 2055) - Impact of rebuild (created/removed/orphaned)
- `preview_revertable_variant_structure_additions()` (Line 2137) - What can be reverted
- `revert_revertable_variant_structure_additions()` (Line 2246) - Revert variant structure to previous state

#### Selection Resolution:
- `resolve_variant_selections_for_variant()` (Line 2491) - Get all option value selections for variant
- `resolve_storefront_variants_for_product_slug()` (Line 2546) - Get variants for storefront display

#### System Suggestions:
- `ensure_system_variant_suggestions()` (Line 1094) - Initialize system variant templates (Size, Color, etc.)

### 5. API Routes (FastAPI Endpoints)
**Location:** [PX-B/app/modules/catalog/router.py](PX-B/app/modules/catalog/router.py)

#### Variant Template Endpoints:
| Method | Endpoint | Handler | Lines | Description |
|--------|----------|---------|-------|-------------|
| GET | `/admin/stores/{store_id}/variant-templates` | `list_variant_templates_admin()` | 578 | List templates for store |
| POST | `/admin/products/{product_id}/variant-templates/{template_id}/apply` | `apply_variant_template_admin()` | 654 | Apply template to product |
| GET | `/admin/products/{product_id}/variant-templates/{template_id}/apply-impact` | `get_variant_template_apply_impact_admin()` | 757 | Preview template application |

#### Variant Management Endpoints:
| Method | Endpoint | Handler | Lines | Description |
|--------|----------|---------|-------|-------------|
| POST | `/admin/products/{product_id}/variants` | `create_product_variant()` | 1247 | Create single variant |
| GET | `/admin/products/{product_id}/variants` | `list_product_variants_admin()` | 1287 | List product variants |
| PATCH | `/admin/products/{product_id}/variants/{variant_id}` | `patch_product_variant()` | 1840 | Update variant fields |
| DELETE | `/admin/products/{product_id}/variants/{variant_id}` | `delete_product_variant_admin()` | 1883 | Delete variant |
| GET | `/admin/products/{product_id}/variants/{variant_id}` | `get_product_variant_admin()` | 1915 | Get variant details |

#### Variant Job Endpoints (Async Operations):
| Method | Endpoint | Handler | Lines | Description |
|--------|----------|---------|-------|-------------|
| POST | `/admin/products/{product_id}/variant-jobs/generate-missing` | `create_generate_missing_variant_job_admin()` | 1328 | Queue generate job (async) |
| POST | `/admin/products/{product_id}/variant-jobs/rebuild` | `create_rebuild_variant_job_admin()` | 1372 | Queue rebuild job (async) |
| GET | `/admin/products/{product_id}/variant-jobs/active` | `get_active_variant_job_admin()` | 1416 | Get active job status |
| GET | `/admin/products/{product_id}/variant-jobs/{job_id}` | `get_variant_job_admin()` | 1442 | Get job details |
| POST | `/admin/products/{product_id}/variants/generate-missing` | `generate_missing_product_variants_admin()` | 1476 | Generate variants (sync) |
| POST | `/admin/products/{product_id}/variants/rebuild` | `rebuild_product_variants_admin()` | 1540 | Rebuild variants (sync) |
| GET | `/admin/products/{product_id}/variants/rebuild-impact` | `get_product_variant_rebuild_impact_admin()` | 1604 | Preview rebuild impact |
| GET | `/admin/products/{product_id}/variants/revert-preview` | `get_product_variant_revert_preview_admin()` | 1629 | Preview revertable changes |
| POST | `/admin/products/{product_id}/variants/revert-additions` | `revert_product_variant_additions_admin()` | 1806 | Revert variant structure |

#### Storefront Variant Endpoints:
| Method | Endpoint | Handler | Lines | Description |
|--------|----------|---------|-------|-------------|
| GET | `/storefront/{store_domain}/products/{product_slug}/variants` | `list_storefront_variants()` | 2442 | Get variants for product display |

#### Helper Functions:
- `_build_variant_read()` (Line 2544) - Convert DB variant to response DTO
- `_build_variant_template_from_rows()` (Line 2574) - Build template from query rows
- `_build_variant_template_read()` (Line 2619) - Convert template to response DTO

**Job Execution Flow:**
- Endpoints use FastAPI's `BackgroundTasks` parameter
- Job created and immediately committed to DB
- Job ID passed to `background_tasks.add_task(execute_catalog_variant_job, job_id)`
- Client receives 202 ACCEPTED with job details immediately
- `execute_catalog_variant_job()` runs async in background, updating job status through phases

---

## Frontend Structure (PX-F)

### 1. Core Type Definitions
**Location:** [PX-F/px/lib/catalog/types.ts](PX-F/px/lib/catalog/types.ts)

#### Variant Types:
- `ProductVariantRead` - Complete variant data with SKU, pricing, stock, status
- `ProductVariantCreateRequest` - Payload for creating variant
- `ProductVariantUpdateRequest` - Payload for updating variant
- `ProductVariantOptionSelectionRead` - Individual option value selection
- `ProductVariantListResponse` - Paginated variants response

#### Job Types:
- `CatalogVariantJobRead` (Line 433) - Job status object
  - Fields: `id`, `product_id`, `type` ("generate_missing_variants" | "rebuild_variants"), `status`, `phase`, `created_count`, `removed_count`, `preserved_count`, `media_rebind_count`, `attribute_delete_count`, `error_message`, timestamps

- `CatalogVariantJobStatus` (Line 431) - "queued" | "running" | "completed" | "failed" | "cancelled"

#### Template Types:
- `VariantTemplateRead` - Template with options and metadata
- `VariantTemplateListResponse` - List of templates
- `VariantTemplateOptionRead` - Option within template
- `VariantTemplateApplyRequest` - Apply template request

#### Impact/Preview Types:
- `ProductVariantRebuildImpactRead` - Rebuild impact analysis (created/removed/orphaned counts)
- `ProductVariantRebuildResponse` - Response after rebuild
- `VariantDeleteImpactRead` - Impact of deleting variant value
- `ProductVariantStructureRevertPreviewRead` - Revertable additions preview
- `ProductVariantStructureRevertResponse` - Revert response

#### Occupancy Types:
- `VariantOccupancyProjection` - Existing variant combination keys
- `AuthoritativeSnapshot` - Full product state snapshot with options, total_variants, active_job, rebuild_impact

### 2. API Client Functions
**Location:** [PX-F/px/lib/catalog/api.ts](PX-F/px/lib/catalog/api.ts)

#### Variant Template APIs:
- `listVariantTemplates()` (Line 181) - GET `/catalog/admin/stores/{store_id}/variant-templates`
- `applyVariantTemplateToProduct()` (Line 194) - POST `/catalog/admin/products/{product_id}/variant-templates/{template_id}/apply`
- `getVariantTemplateApplyImpact()` (Line 232) - GET `/catalog/admin/products/{product_id}/variant-templates/{template_id}/apply-impact`

#### Variant CRUD APIs:
- `createProductVariant()` (Line 522) - POST `/catalog/admin/products/{product_id}/variants`
- `listProductVariants()` (Line 534) - GET `/catalog/admin/products/{product_id}/variants`
- `updateProductVariant()` (Line 552) - PATCH `/catalog/admin/products/{product_id}/variants/{variant_id}`
- `deleteProductVariant()` (Line 565) - DELETE `/catalog/admin/products/{product_id}/variants/{variant_id}`

#### Variant Job APIs:
- `createGenerateMissingVariantJob()` (Line 605) - POST `/catalog/admin/products/{product_id}/variant-jobs/generate-missing`
- `createRebuildVariantJob()` (Line 613) - POST `/catalog/admin/products/{product_id}/variant-jobs/rebuild`
- `getActiveVariantJob()` (Line 621) - GET `/catalog/admin/products/{product_id}/variant-jobs/active`
- `getVariantJob()` (Line 629) - GET `/catalog/admin/products/{product_id}/variant-jobs/{job_id}`
- `generateMissingProductVariants()` (Line 597) - POST `/catalog/admin/products/{product_id}/variants/generate-missing` (sync)
- `rebuildProductVariants()` (Line 589) - POST `/catalog/admin/products/{product_id}/variants/rebuild` (sync)

#### Variant Impact/Preview APIs:
- `getProductVariantRebuildImpact()` (Line 637) - GET `/catalog/admin/products/{product_id}/variants/rebuild-impact`
- `getProductVariantRevertPreview()` (Line 662) - GET `/catalog/admin/products/{product_id}/variants/revert-preview`
- `revertProductVariantAdditions()` (Line 670) - POST `/catalog/admin/products/{product_id}/variants/revert-additions`

#### Storefront APIs:
- `listStorefrontProductVariants()` (Line 678) - GET `/storefront/{store_domain}/products/{product_slug}/variants`

### 3. Variant Workspace (State Models)
**Location:** [PX-F/px/lib/catalog/product-variant-workspace.ts](PX-F/px/lib/catalog/product-variant-workspace.ts)

#### Type Definitions:
- `VariantDraft` - Editable variant fields (SKU, price, stock, weight, active)
- `OptionEditorDraft` - Editable option fields (name, display_type)
- `ValueEditorDraft` - Editable value fields (value, color_hex, image_url)
- `ValueDraftVisual` - Visual representation (color_hex, image_url)

#### Utility Functions:
- `toVariantDraft()` - Convert ProductVariantRead to editable VariantDraft
- `getSelectedOption()` - Get active option from list
- `getSelectedOptionDraft()` - Get draft data for selected option
- `getSelectedValueId()` - Get active value ID for option

### 4. Variant Combination Matrix
**Location:** [PX-F/px/lib/catalog/variant-matrix.ts](PX-F/px/lib/catalog/variant-matrix.ts)

#### Key Types:
- `VariantCombinationRow` - Single combination: selections + matching variant
- `VariantCombinationState` - Full matrix state including summary counts

#### Matrix Calculation Functions:
- `combinationKeyFromIds()` - Generate sorted combination key
- `variantKeyFromSelections()` - Generate key from selections
- `getVariantCombinationState()` - Main function computing:
  - `optionCount`: Total options
  - `completeOptionCount`: Options with values
  - `isReady`: All options have values
  - `possibleCount`: Total possible combinations (product of value counts)
  - `currentCount`: Existing variants
  - `missingCount`: Possible but not created
  - `orphanedCount`: Variants with invalid combinations
  - `rows`: Array of VariantCombinationRow (limited to 200)
  - `rowsAreTruncated`: Boolean if > 200 possible combinations

#### Constants:
- `VARIANT_COMBINATION_PREVIEW_LIMIT = 200` - Max rows to display

### 5. Variant Actions Hook
**Location:** [PX-F/px/app/components/product-editor/use-product-variant-actions.ts](PX-F/px/app/components/product-editor/use-product-variant-actions.ts)

**Hook Parameters:**
```typescript
type UseProductVariantActionsParams = {
  productId: string;
  onStateChange?: (state: OperationalState) => void;
  onJobStarted?: (job: CatalogVariantJobRead) => void;
  onJobProgress?: (job: CatalogVariantJobRead) => void;
  onJobCompleted?: (job: CatalogVariantJobRead) => void;
  onJobFailed?: (job: CatalogVariantJobRead, error: string) => void;
  polling?: { interval: number; maxAttempts: number };
}
```

**Main Actions Provided:**
- `createVariant()` - Create new variant with option selections
- `updateVariant()` - Update variant fields (SKU, price, stock, weight, active)
- `deleteVariant()` - Delete variant with confirmation
- `generateMissingVariants()` - Trigger async job to generate all missing combinations
- `rebuildVariants()` - Trigger async job to rebuild entire variant set
- `revertVariantAdditions()` - Revert to previous variant structure

**Job Polling:**
- `startPollingVariantJob()` - Poll job status until completion/failure
- Supports callbacks for progress updates
- Default: 1s interval, 600 attempts (10 minutes max)

**Impact Analysis:**
- `getVariantDeleteImpact()` - Analyze impact of deleting variant value
- `getVariantRebuildImpact()` - Preview rebuild changes
- `getRevertPreview()` - Preview what can be reverted

### 6. Variant Structure Studio Component
**Location:** [PX-F/px/app/components/product-editor/variant-structure-studio.tsx](PX-F/px/app/components/product-editor/variant-structure-studio.tsx)

**Props:**
```typescript
type Props = {
  locale: Locale;
  copy: CatalogAdminCopy["productEdit"];
  options: ProductOptionRead[];
  templates: VariantTemplateRead[];
  selectedTemplateId: string;
  templateApplying: boolean;
  copySourceId?: string;
  copySearchOptions?: ProductSearchResultRead[];
  copyingStructure?: boolean;
  selectedOption: ProductOptionRead | null;
  selectedValueId: string;
  busyVariants: boolean;
  combinationState: VariantCombinationState;
  combinationSummary: string;
  missingCombinationSummary: string;
  selectedValueDeleteImpact: VariantDeleteImpactRead | null;
  rebuildImpact: ProductVariantRebuildImpactRead | null;
  activeVariantJob?: CatalogVariantJobRead | null;
  operationalState: OperationalState;
  formatLocaleLabel: (localeCode: string) => string;
  
  // Event handlers
  onClose: () => void;
  onSelectOption: (optionId: string) => void;
  onSelectTemplate: (templateId: string) => void;
  onApplyTemplate: () => void;
  onSearchCopyProducts?: (query: string) => void;
  onSelectCopySource?: (productId: string) => void;
  onCopyStructure?: () => void;
  onOpenCreateOptionModal: () => void;
  onMoveOption: (optionId: string, direction: -1 | 1) => void;
  onOpenOptionTranslations: () => void;
  onDeleteOption: (optionId: string) => void;
  onDeleteOptionValue: (optionValueId: string) => void;
  onSelectOptionValue: (optionId: string, valueId: string) => void;
  onOpenValueManager: (optionId: string, valueId: string) => void;
  onOpenAddValueModal: () => void;
  onGenerateVariants: () => void;
  onResetVariants: (options?: { skipConfirm?: boolean }) => void;
}
```

**Features:**
- Template selection and application
- Option/value management with reordering
- Structure copy from another product
- Combination matrix visualization (limited to 200 rows)
- Missing combinations display
- Rebuild action with impact preview
- Variant job status tracking
- Repair actions (show/hide based on state)

### 7. Related Components
**Location:** [PX-F/px/app/components/product-editor/](PX-F/px/app/components/product-editor/)

- `product-variants-section.tsx` - Main variants section component
- `variant-table-row.tsx` - Individual variant row renderer
- `product-add-option-modal.tsx` - Modal for creating options
- `product-add-value-modal.tsx` - Modal for adding values to option
- `product-value-management-drawer.tsx` - Drawer for editing value details
- `product-option-translation-drawer.tsx` - Drawer for translating option names
- `variant-empty-state.tsx` - Empty state UI when no options
- `shared.tsx` - Shared components (ReorderControl, ValueVisual, etc.)
- `operational-hud.tsx` - Operational state HUD

### 8. Test Files
**Location:** [PX-F/px/lib/catalog/__tests__/](PX-F/px/lib/catalog/__tests__/)

- `variant-matrix.test.ts` - Combination matrix calculations
- `product-variant-workspace.test.ts` - Workspace utilities and conversions
- `api.test.ts` - API function tests
- `validation.test.ts` - Variant validation

**Location:** [PX-F/px/app/components/product-editor/__tests__/](PX-F/px/app/components/product-editor/__tests__/)

- `variant-structure-studio.test.tsx` - Component tests
- `product-variants-section.test.tsx` - Section component tests

---

## Backend Test Files (PX-B)

**Location:** [PX-B/tests/](PX-B/tests/)

| File | Purpose | Key Tests |
|------|---------|-----------|
| `test_catalog_variants.py` | Core variant operations | Create options/values/variants, CRUD operations |
| `test_catalog_variant_sync.py` | Variant synchronization | Data consistency, variant rebuilds |
| `test_catalog_snapshot.py` | Snapshot creation & validation | Product structure snapshots with variant data |
| `test_catalog_snapshot_integrity.py` | Snapshot integrity | Variants in lightweight vs full mode, job execution |

---

## Key Data Flow Diagrams

### Variant Generation Flow (Backend):
```
1. User Request
   ↓
2. create_generate_missing_variant_job_admin() (Router)
   ↓
3. create_catalog_variant_job_for_product() (Service)
   → Validates structure guard (version/hash match)
   → Creates CatalogVariantJob record
   → Returns job to client
   ↓
4. background_tasks.add_task() (Router)
   → Schedules execute_catalog_variant_job(job_id)
   ↓
5. execute_catalog_variant_job() (Service) - Async
   → Phase: VALIDATING → structure guard re-check
   → Phase: COMPUTING_COMBINATIONS → calculate possible variants
   → Phase: CREATING_VARIANTS → insert new VariantValueLink rows
   → Phase: FINALIZING → update counts
   → Status: COMPLETED
   ↓
6. Client polls getActiveVariantJob() or getVariantJob()
   → Gets job status, phase, progress counts
```

### Variant Template Application Flow (Backend):
```
1. apply_variant_template_admin() (Router)
   ↓
2. apply_variant_template_to_product() (Service)
   → Get template with options
   → For each template option:
     → Create ProductOption (if not exists)
     → Add ProductOptionValues from template
   → Optionally trigger variant generation
   ↓
3. Return updated product with options
```

### Frontend State Management:
```
VariantStructureStudio Component
   ↓
use-product-variant-actions.ts Hook
   ├─ State: combinationState (VariantCombinationState)
   ├─ State: activeVariantJob (CatalogVariantJobRead | null)
   ├─ State: operationalState (INITIALIZING|SYNCED|EDITING|PROCESSING|FAILED)
   ├─ Actions: createVariant, updateVariant, deleteVariant
   ├─ Actions: generateMissingVariants, rebuildVariants, revertVariants
   └─ Job Polling: startPollingVariantJob()
      → Monitors job status transitions
      → Updates UI based on phase
      → Handles completion/failure callbacks
```

---

## Key Architectural Patterns

### 1. Structure Guard Mechanism
- Every variant operation includes `expected_version` and `expected_hash`
- Backend validates these match before proceeding
- Prevents stale/out-of-sync operations
- Applied in: template application, variant generation, revert operations

### 2. Async Job Execution
- Long-running operations (generate/rebuild) use async jobs
- FastAPI BackgroundTasks for job scheduling
- Database stores job state for status polling
- Phase tracking for progress indication

### 3. Combination Key Generation
- Deterministic: `sorted(option_value_ids).join("|")`
- Used for duplicate detection
- Enables orphaned variant detection
- Immutable for given option/value combination

### 4. Occupancy Projection
- Tracks existing variant combinations
- Enables missing/orphaned detection
- Frontend highlights unimplemented combinations

### 5. Impact Analysis
- Preview operations before executing
- Calculate created/removed/preserved counts
- Identify cleanup needs (media, attributes)
- Used for confirmation dialogs

---

## Common Operations

### Creating a Variant:
```
POST /catalog/admin/products/{product_id}/variants
{
  "combination_key": "val1|val2|val3",
  "sku": "PROD-SKU-001",
  "price_override": "99.99",
  "stock_qty": 100,
  "weight": "1.5",
  "is_active": true,
  "selections": [
    { "option_value_id": "uuid-1" },
    { "option_value_id": "uuid-2" },
    { "option_value_id": "uuid-3" }
  ]
}
```

### Generating Missing Variants (Async):
```
POST /catalog/admin/products/{product_id}/variant-jobs/generate-missing
{
  "expected_version": 5,
  "expected_hash": "abc123def456"
}
```
Response: `202 ACCEPTED` with `CatalogVariantJobRead` containing job_id for polling

### Applying Variant Template:
```
POST /catalog/admin/products/{product_id}/variant-templates/{template_id}/apply
{
  "expected_version": 5,
  "expected_hash": "abc123def456"
}
```
Creates options and values from template definition

### Checking Job Status:
```
GET /catalog/admin/products/{product_id}/variant-jobs/{job_id}
```
Returns: `CatalogVariantJobRead` with current phase, counts, status

---

## Integration Points

### Frontend-Backend Communication:
1. **Initialization**: Fetch product snapshot including variant data
2. **Template List**: Load available templates for store
3. **Template Apply**: POST apply with structure guard
4. **Variant CRUD**: Direct variant creation/update/delete
5. **Job Operations**: POST job creation, then polling via GET
6. **Impact Preview**: GET before destructive operations

### Database Dependencies:
- Variant operations depend on: Products, ProductOptions, ProductOptionValues
- Jobs track: Product state, User who initiated, Operation type
- Templates reference: OptionLibraryOptions globally or per-store

### Type Safety:
- TypeScript frontend mirrors backend Pydantic schemas
- API client wraps all endpoints with type safety
- Test files validate type contracts
