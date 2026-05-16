# i18n Implementation Specification

**Task**: Implement multi-language internationalization framework  
**Priority**: 🔴 BLOCKER  
**Estimated Effort**: 4-5 hours  
**Files**: Backend errors + Frontend UI strings

---

## Phase 1: Backend i18n Keys

### Step 1: Create `PX-B/app/core/i18n_keys.py`

**Purpose**: Central registry of all translatable message keys

```python
"""
Centralized i18n message keys for the entire application.
All user-facing error messages must use a key from this module.

Key Naming Convention:
  {domain}.{feature}.{type}.{specific}
  
Examples:
  - auth.login.errors.invalid_credentials
  - catalog.variant.errors.explosion_risk
  - catalog.product.ui.save_success
"""

# ============================================================
# CATALOG VARIANT DOMAIN
# ============================================================

class CatalogVariantErrorKeys:
    """Error message keys for variant operations"""
    
    # Job lifecycle errors
    ACTIVE_JOB_EXISTS = "catalog.variant.errors.active_job_exists"
    JOB_NOT_FOUND = "catalog.variant.errors.job_not_found"
    
    # Structure validation errors
    STRUCTURE_STALE_VERSION = "catalog.variant.errors.structure_stale_version"
    STRUCTURE_STALE_HASH = "catalog.variant.errors.structure_stale_hash"
    STRUCTURE_INVALID = "catalog.variant.errors.structure_invalid"
    
    # Explosion protection errors
    EXPLOSION_RISK = "catalog.variant.errors.explosion_risk"
    TOO_MANY_OPTIONS = "catalog.variant.errors.too_many_options"
    TOO_MANY_VALUES = "catalog.variant.errors.too_many_values"
    TIER_QUOTA_EXCEEDED = "catalog.variant.errors.tier_quota_exceeded"
    
    # Variant errors
    INVALID_COMBINATION = "catalog.variant.errors.invalid_combination"
    DUPLICATE_SKU = "catalog.variant.errors.duplicate_sku"
    VARIANT_NOT_FOUND = "catalog.variant.errors.variant_not_found"
    
    # Worker errors
    JOB_TIMEOUT = "catalog.variant.errors.job_timeout"
    JOB_CANCELLED = "catalog.variant.errors.job_cancelled"
    GENERATION_FAILED = "catalog.variant.errors.generation_failed"
    
    # Template errors
    TEMPLATE_NOT_FOUND = "catalog.variant.errors.template_not_found"
    TEMPLATE_APPLICATION_FAILED = "catalog.variant.errors.template_application_failed"

class CatalogVariantUIKeys:
    """UI string keys for variant operations"""
    
    # Job status labels
    STATUS_QUEUED = "catalog.variant.ui.status.queued"
    STATUS_RUNNING = "catalog.variant.ui.status.running"
    STATUS_COMPLETED = "catalog.variant.ui.status.completed"
    STATUS_FAILED = "catalog.variant.ui.status.failed"
    STATUS_CANCELLED = "catalog.variant.ui.status.cancelled"
    STATUS_TIMEOUT = "catalog.variant.ui.status.timeout"
    
    # Action buttons
    GENERATE_MISSING = "catalog.variant.ui.buttons.generate_missing"
    REBUILD_ALL = "catalog.variant.ui.buttons.rebuild_all"
    CANCEL_JOB = "catalog.variant.ui.buttons.cancel_job"
    APPLY_TEMPLATE = "catalog.variant.ui.buttons.apply_template"
    
    # Progress messages
    PROGRESS_CREATED = "catalog.variant.ui.progress.created"
    PROGRESS_REMOVED = "catalog.variant.ui.progress.removed"
    PROGRESS_PRESERVED = "catalog.variant.ui.progress.preserved"
    
    # Notification messages
    JOB_STARTED = "catalog.variant.ui.notifications.job_started"
    JOB_COMPLETED = "catalog.variant.ui.notifications.job_completed"
    JOB_FAILED = "catalog.variant.ui.notifications.job_failed"

# Convenience exports
CATALOG_VARIANT_ERROR_KEYS = CatalogVariantErrorKeys()
CATALOG_VARIANT_UI_KEYS = CatalogVariantUIKeys()

# ============================================================
# PRODUCT DOMAIN
# ============================================================

class CatalogProductErrorKeys:
    """Error message keys for product operations"""
    
    NOT_FOUND = "catalog.product.errors.not_found"
    SLUG_DUPLICATE = "catalog.product.errors.slug_duplicate"
    INVALID_STATUS = "catalog.product.errors.invalid_status"
    MISSING_TRANSLATION = "catalog.product.errors.missing_translation"

class CatalogProductUIKeys:
    """UI strings for product operations"""
    
    CREATE_SUCCESS = "catalog.product.ui.create_success"
    UPDATE_SUCCESS = "catalog.product.ui.update_success"
    DELETE_SUCCESS = "catalog.product.ui.delete_success"

# Add more domains as needed...
```

### Step 2: Update Error Classes

Modify `PX-B/app/exceptions/errors.py`:

```python
"""
Update all error classes to accept optional message_key parameter
"""

from app.core.i18n_keys import CATALOG_VARIANT_ERROR_KEYS

class CatalogVariantJobActiveError(HTTPException):
    def __init__(self, message_key: str = None, details: dict = None):
        self.message_key = message_key or CATALOG_VARIANT_ERROR_KEYS.ACTIVE_JOB_EXISTS
        self.details = details or {}
        super().__init__(
            status_code=status.HTTP_409_CONFLICT,
            detail={
                "key": self.message_key,
                "details": self.details
            }
        )

class ProductStructureStaleVersionError(HTTPException):
    def __init__(self, message_key: str = None):
        self.message_key = message_key or CATALOG_VARIANT_ERROR_KEYS.STRUCTURE_STALE_VERSION
        super().__init__(
            status_code=status.HTTP_409_CONFLICT,
            detail={"key": self.message_key}
        )

# ... etc for all error classes
```

### Step 3: Update Router Error Responses

File: `PX-B/app/modules/catalog/router.py`

```python
from app.core.i18n_keys import CATALOG_VARIANT_ERROR_KEYS, CATALOG_VARIANT_UI_KEYS

@router.post("/admin/products/{product_id}/variant-jobs/generate-missing")
async def create_generate_missing_variant_job_admin(
    product_id: UUID,
    current_user: ActiveUser = Depends(),
    session: Session = Depends(get_session),
):
    """
    When errors occur, return i18n key instead of message.
    Frontend will translate using the key.
    """
    try:
        product = get_product_by_id(session, product_id)
        if not product:
            raise HTTPException(
                status_code=404,
                detail={
                    "error": "not_found",
                    "i18n_key": "catalog.product.errors.not_found"
                }
            )
        
        # Check for active job
        active_job = get_active_catalog_variant_job_for_product(session, product_id)
        if active_job:
            raise HTTPException(
                status_code=409,
                detail={
                    "error": "active_job_exists",
                    "i18n_key": CATALOG_VARIANT_ERROR_KEYS.ACTIVE_JOB_EXISTS,
                    "context": {"product_id": str(product_id)}
                }
            )
        
        # ... rest of logic
        
    except HTTPException:
        raise
    except Exception as e:
        logger.error(f"Error creating job: {e}")
        raise HTTPException(
            status_code=500,
            detail={
                "error": "generation_failed",
                "i18n_key": CATALOG_VARIANT_ERROR_KEYS.GENERATION_FAILED
            }
        )
```

---

## Phase 2: Create Translation Files

### Step 1: Create Directory Structure

```bash
mkdir -p PX-F/px/public/locales/{en,es,fr,de}
touch PX-F/px/public/locales/en/translations.json
touch PX-F/px/public/locales/es/translations.json
touch PX-F/px/public/locales/fr/translations.json
touch PX-F/px/public/locales/de/translations.json
```

### Step 2: Create English Base Translation

File: `PX-F/px/public/locales/en/translations.json`

```json
{
  "catalog": {
    "variant": {
      "errors": {
        "active_job_exists": "A variant generation job is already running for this product. Please wait for it to complete before starting another.",
        "job_not_found": "The job you're looking for doesn't exist or has expired.",
        "structure_stale_version": "The product structure was modified. Please refresh and try again.",
        "structure_stale_hash": "The product structure has changed since you started. Please refresh the page.",
        "structure_invalid": "The product structure is invalid. Please review and fix before proceeding.",
        "explosion_risk": "Too many variants would be created ({{count}} total). This exceeds the limit for your plan ({{limit}}). Try reducing options or values.",
        "too_many_options": "Products can have a maximum of 10 options. You tried to add {{count}}.",
        "too_many_values": "Each option can have a maximum of 100 values. This option has {{count}}.",
        "tier_quota_exceeded": "Your {{tier}} plan allows up to {{limit}} variants. You would create {{projected}} with these changes.",
        "invalid_combination": "This combination of option values is invalid or duplicate.",
        "duplicate_sku": "The SKU '{{sku}}' is already in use for this product.",
        "variant_not_found": "The variant you're looking for doesn't exist.",
        "job_timeout": "The variant job timed out after {{duration}} minutes of inactivity.",
        "job_cancelled": "The variant job was cancelled by the user.",
        "generation_failed": "Variant generation failed. Please try again or contact support."
      },
      "ui": {
        "status": {
          "queued": "Queued",
          "running": "Running",
          "completed": "Completed",
          "failed": "Failed",
          "cancelled": "Cancelled",
          "timeout": "Timeout"
        },
        "buttons": {
          "generate_missing": "Generate Missing Variants",
          "rebuild_all": "Rebuild All Variants",
          "cancel_job": "Cancel Job",
          "apply_template": "Apply Template"
        },
        "progress": {
          "created": "{{count}} variant created",
          "created_plural": "{{count}} variants created",
          "removed": "{{count}} variant removed",
          "removed_plural": "{{count}} variants removed",
          "preserved": "{{count}} variant preserved",
          "preserved_plural": "{{count}} variants preserved"
        },
        "notifications": {
          "job_started": "Variant generation started",
          "job_completed": "Variant generation completed successfully",
          "job_failed": "Variant generation failed"
        }
      }
    },
    "product": {
      "errors": {
        "not_found": "Product not found",
        "slug_duplicate": "A product with this slug already exists in this store",
        "invalid_status": "Invalid product status",
        "missing_translation": "Product must have a translation in the default language"
      },
      "ui": {
        "create_success": "Product created successfully",
        "update_success": "Product updated successfully",
        "delete_success": "Product deleted successfully"
      }
    }
  }
}
```

### Step 3: Create Spanish Translation

File: `PX-F/px/public/locales/es/translations.json`

```json
{
  "catalog": {
    "variant": {
      "errors": {
        "active_job_exists": "Un trabajo de generación de variantes ya está en ejecución para este producto. Por favor, espere a que se complete antes de iniciar otro.",
        "job_not_found": "El trabajo que busca no existe o ha expirado.",
        "structure_stale_version": "La estructura del producto se modificó. Por favor, actualice e intente de nuevo.",
        "structure_stale_hash": "La estructura del producto ha cambiado desde que comenzó. Por favor, actualice la página.",
        "structure_invalid": "La estructura del producto no es válida. Por favor, revise y corrija antes de continuar.",
        "explosion_risk": "Se crearían demasiadas variantes ({{count}} total). Esto supera el límite de su plan ({{limit}}). Intente reducir opciones o valores.",
        "too_many_options": "Los productos pueden tener un máximo de 10 opciones. Intentó agregar {{count}}.",
        "too_many_values": "Cada opción puede tener un máximo de 100 valores. Esta opción tiene {{count}}.",
        "tier_quota_exceeded": "Su plan {{tier}} permite hasta {{limit}} variantes. Crearían {{projected}} con estos cambios."
      },
      "ui": {
        "status": {
          "queued": "En cola",
          "running": "En ejecución",
          "completed": "Completado",
          "failed": "Falló",
          "cancelled": "Cancelado",
          "timeout": "Tiempo agotado"
        },
        "buttons": {
          "generate_missing": "Generar Variantes Faltantes",
          "rebuild_all": "Reconstruir Todas las Variantes",
          "cancel_job": "Cancelar Trabajo",
          "apply_template": "Aplicar Plantilla"
        }
      }
    }
  }
}
```

---

## Phase 3: Frontend i18n Integration

### Step 1: Install Dependencies

```bash
cd PX-F/px
npm install i18next react-i18next --save
npm install i18next-browser-languagedetector --save  # Optional: auto-detect browser language
```

### Step 2: Create i18n Config

File: `PX-F/px/lib/i18n/index.ts`

```typescript
import i18n from 'i18next';
import { initReactI18next } from 'react-i18next';
import LanguageDetector from 'i18next-browser-languagedetector';

// Import translation files
import enTranslations from '@/public/locales/en/translations.json';
import esTranslations from '@/public/locales/es/translations.json';
import frTranslations from '@/public/locales/fr/translations.json';
import deTranslations from '@/public/locales/de/translations.json';

const resources = {
  en: { translation: enTranslations },
  es: { translation: esTranslations },
  fr: { translation: frTranslations },
  de: { translation: deTranslations },
};

i18n
  .use(LanguageDetector)
  .use(initReactI18next)
  .init({
    resources,
    fallbackLng: 'en',
    defaultNS: 'translation',
    ns: ['translation'],
    interpolation: {
      escapeValue: false, // React already prevents XSS
    },
    detection: {
      order: ['localStorage', 'navigator'],
      caches: ['localStorage'],
    },
  });

export default i18n;
```

### Step 3: Initialize in App

File: `PX-F/px/app/layout.tsx`

```typescript
import i18n from '@/lib/i18n';

// Initialize i18n when app loads
if (typeof window !== 'undefined' && !i18n.isInitialized) {
  i18n.init();
}

export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <html lang={i18n.language}>
      <body>{children}</body>
    </html>
  );
}
```

### Step 4: Update Error Handling Components

File: `PX-F/px/lib/catalog/error-handler.ts`

```typescript
import { useTranslation } from 'react-i18next';

export function useVariantErrorHandler() {
  const { t } = useTranslation();

  return {
    getErrorMessage(error: any): string {
      // If it's an i18n key response from backend
      if (error.response?.data?.i18n_key) {
        const key = error.response.data.i18n_key;
        const context = error.response.data.context || {};
        return t(key, context);
      }

      // Fallback for unexpected errors
      return t('catalog.variant.errors.generation_failed');
    },

    getErrorDetails(error: any): Record<string, any> {
      return error.response?.data?.context || {};
    },
  };
}
```

### Step 5: Update Components to Use i18n

File: `PX-F/px/app/components/VariantStructureStudio.tsx`

```typescript
import { useTranslation } from 'react-i18next';
import { useVariantErrorHandler } from '@/lib/catalog/error-handler';

export function VariantStructureStudio() {
  const { t } = useTranslation();
  const { getErrorMessage } = useVariantErrorHandler();

  const handleJobCreation = async () => {
    try {
      // ... API call
    } catch (error) {
      const message = getErrorMessage(error);
      toast.error(message);
    }
  };

  return (
    <div>
      <button onClick={handleJobCreation}>
        {t('catalog.variant.ui.buttons.generate_missing')}
      </button>
      
      {jobStatus && (
        <div>
          {t(`catalog.variant.ui.status.${jobStatus.toLowerCase()}`)}
        </div>
      )}
    </div>
  );
}
```

---

## Testing Checklist

### Backend Tests

```bash
# Verify all error keys are used
grep -r "CATALOG_VARIANT_ERROR_KEYS" PX-B/app/

# Run tests
cd PX-B && python -m pytest tests/ -v -k "i18n or error"
```

**Test Code Pattern:**
```python
def test_active_job_error_returns_i18n_key(client, session):
    # Setup product with active job
    job = CatalogVariantJob(...)
    session.add(job)
    session.commit()
    
    # Try to create another job
    response = client.post(
        f"/catalog/admin/products/{product_id}/variant-jobs/generate-missing"
    )
    
    assert response.status_code == 409
    assert response.json()["i18n_key"] == "catalog.variant.errors.active_job_exists"
```

### Frontend Tests

```bash
# Verify translations are loaded
cd PX-F/px && npm test -- --grep "i18n"
```

**Test Code Pattern:**
```typescript
test('error message uses i18n key', async () => {
  const { result } = renderHook(() => useVariantErrorHandler());
  
  const error = {
    response: {
      data: {
        i18n_key: 'catalog.variant.errors.active_job_exists',
        context: { product_id: '123' },
      },
    },
  };
  
  const message = result.current.getErrorMessage(error);
  expect(message).toBe('A variant generation job is already running...');
});
```

### Validation Commands

```bash
# All error keys used
grep -r "i18n_key" PX-B/app/ | wc -l

# All translation keys present
cd PX-F/px && npm run i18n:check

# No hardcoded error strings in UI
grep -r "error message" PX-F/px/app/components/ | grep -v i18n | wc -l
```

---

## Completion Checklist

- [ ] All error classes updated with `i18n_key`
- [ ] All router endpoints return `i18n_key` in error responses
- [ ] Translation files created for EN, ES, FR, DE
- [ ] Frontend i18n config initialized
- [ ] All error messages use `t()` hook
- [ ] Backend tests verify i18n keys
- [ ] Frontend tests verify translations load
- [ ] `npm run i18n:check` passes
- [ ] No hardcoded error strings remain

---

**Status**: Ready for implementation  
**Owner**: Code Agent  
**Due**: As part of BLOCKER items
