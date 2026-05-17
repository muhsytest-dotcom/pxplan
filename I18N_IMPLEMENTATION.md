# i18n Implementation Reference

Last updated: 2026-05-17

## Status

Implemented. This is no longer a blocker task.

Backend:
- `PX-B/app/core/i18n_keys.py`
- `PX-B/app/exceptions/errors.py`
- `PX-B/app/modules/catalog/router.py`

Frontend:
- `PX-F/lib/catalog/i18n.ts`
- `PX-F/lib/i18n/ui-copy.ts`
- Existing locale/path routing and copy infrastructure.

Do not install or migrate to a second frontend i18n framework. The project currently uses its own established i18n approach and `npm run i18n:check` validates 10 locales.

## Current Contract

Backend user-facing errors should include:

```json
{
  "error": {
    "code": "SOME_CODE",
    "i18n_key": "catalog.variant.errors.some_key",
    "context": {}
  }
}
```

Frontend catalog code should preserve and translate backend `i18n_key` values through the existing catalog helper.

## Rules For New Work

- Add backend keys to `PX-B/app/core/i18n_keys.py` when needed.
- Add frontend copy to the existing UI/catalog copy system for all supported locales.
- Preserve fallback behavior for missing translations.
- Use interpolation context instead of hardcoded composed strings where users see variable values.
- Run `npm run i18n:check`.

## Validation

```powershell
cd PX-F
npm run i18n:check
npm test
npm run typecheck
```

Backend error-envelope regressions should be covered with targeted pytest tests for any new API behavior.
