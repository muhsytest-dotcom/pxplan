# New Guides Summary

Last updated: 2026-05-17

## Summary

The new guide system has been realigned to the live PX implementation through Slice 14.

The original guide set was created to help agents implement blocker phases. Those phases are complete. The guide system now directs agents toward release hardening:
- Archive-vs-delete migration.
- Remaining E2E/API coverage.
- Dead-code cleanup.
- Optional observability improvements.

## Start Here

1. `AGENT_HANDOFF.md`
2. `WHAT-DONE.md`
3. `00_START_HERE.md`
4. `CODE_AGENT_COMPREHENSIVE_GUIDE.md`
5. `QUICK_REFERENCE.md`
6. `DO_NOT_FORGET.md`

## Current Warning

Do not follow obsolete historical instructions that claim early blocker work remains, tell agents to start with initial error-classification phases, ask for a separate frontend i18n framework, or ask agents to create files that already exist in the live codebase.

Those statements are historical and have been corrected in the active docs.
