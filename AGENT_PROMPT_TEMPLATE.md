# Agent Prompt Template

Last updated: 2026-05-17

Use this template for future agents. It is aligned with the live system through Slice 14.

```text
You are a code agent working on the PX project.

Before coding:
1. Read pxplan/AGENT_HANDOFF.md.
2. Read pxplan/WHAT-DONE.md.
3. Read pxplan/00_START_HERE.md.
4. Read pxplan/CODE_AGENT_COMPREHENSIVE_GUIDE.md.
5. Keep pxplan/QUICK_REFERENCE.md open.
6. Review pxplan/DO_NOT_FORGET.md before finishing.

Current status:
- The variants stabilization work is complete through Slice 14.
- Error classification, worker retry, i18n, durable SSE, snapshots, metrics, SSE fallback, batched listing resolution, tier enforcement, and published-snapshot storefront regression coverage are already done.
- Do not redo old Phase 1-5 blocker work.

Your task:
[Describe exactly one release-hardening slice.]

Recommended current work:
- Archive-vs-delete migration strategy and implementation.
- Remaining E2E/API coverage for template quota failures, media rebinding after rebuild, and full browser admin-to-storefront flows.
- Dead-code cleanup after archive semantics settle.
- Optional deeper observability.

Implementation rules:
- Follow existing architecture and patterns.
- Preserve backward compatibility unless explicitly changing it through a migration.
- Preserve i18n error envelopes.
- Preserve active-job guards.
- Preserve storefront published snapshot boundaries.
- Add focused tests for all changed behavior.
- Run backend and frontend quality gates when relevant.

After completion:
- Update pxplan/WHAT-DONE.md.
- Update pxplan/AGENT_HANDOFF.md or a dedicated handoff file.
- Include implementation status, completed work, reasoning, architecture decisions, pending tasks, known issues, changed files, API/database/schema changes, assumptions, recommended next steps, and avoid-breaking notes.
```

## Standard Validation Commands

Backend:

```powershell
cd PX-B
.\.venv\bin\python -m pytest tests -q
.\.venv\bin\ruff check .
.\.venv\bin\python -m mypy app
```

Frontend:

```powershell
cd PX-F
npm test
npm run lint
npm run typecheck
npm run i18n:check
npm run build
```
