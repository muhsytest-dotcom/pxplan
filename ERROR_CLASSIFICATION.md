# Error Classification Reference

Last updated: 2026-05-17

## Status

Implemented. This is no longer a blocker task.

Live code:
- `PX-B/app/modules/catalog/job_errors.py`
- `PX-B/app/modules/catalog/worker.py`
- `PX-B/app/modules/catalog/service.py`
- `PX-B/app/modules/catalog/models.py`
- Alembic migration `b8f7a2c9d401_add_variant_job_error_classification.py`

Tests:
- `PX-B/tests/test_job_error_classification.py`
- Related catalog job tests in `PX-B/tests/test_catalog_variants.py` and `PX-B/tests/test_catalog_variant_ops.py`

## Current Behavior

- Job errors are classified by `JobErrorType`.
- Retryable failures use bounded worker retry.
- Non-retryable validation, conflict, cancellation, and stale-structure failures fail fast.
- Job records track `error_type`, `retry_count`, `last_error_at`, and `last_error_message`.
- Failed job events expose stored error classification.
- Timeout cleanup marks stale running jobs as `timeout`.

## Do Not Reimplement

Do not create a second error classification system or change the error envelope shape. Extend the existing classes and tests if a new error type is truly needed.

## Validation

```powershell
cd PX-B
.\.venv\bin\python -m pytest tests/test_job_error_classification.py -q
.\.venv\bin\python -m pytest tests/test_catalog_variants.py tests/test_catalog_variant_ops.py -q
.\.venv\bin\ruff check .
.\.venv\bin\python -m mypy app
```
