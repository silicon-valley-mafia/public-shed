# Issue 06A — Verify granular failed-upload retry end to end

## Goal

Add/complete end-to-end regression coverage for individual, folder-level and selected-batch retry, and verify the safety/lifecycle boundaries.

## Depends on

- All implementation issues in Phases 01–05.

## Scope

Create a realistic folder-import scenario with 100+ files across nested folders and scattered failures. Verify:

- One failed file can be retried alone.
- One containing folder can be retried without sibling folders.
- Retry All affects only the selected batch.
- Successful files are never resent.
- Duplicate filenames in different folders select correctly.
- Changed sources require confirmation.
- Missing sources remain blocked.
- Unknown outcomes and projection_pending are not retried.
- A second failure can be retried again only when safe.
- Pagination and filters do not change selected files.
- Queue-full, wrong-drive, cancellation and concurrent retry behavior are correct.
- Restored Activity history remains display-only.
- Mobile folder import and extracted archive behavior remain accurately documented.

## Acceptance criteria

1. Playwright covers all three retry granularities.
2. Backend tests prove remote accepted/unknown sends are not duplicated.
3. App tests prove original-drive and transfer-busy guards.
4. Frontend tests prove per-file state and stable identity.
5. Existing regular-file bulk retry remains green.
6. Existing upload queue, folder merge, encryption, resumable upload and Activity history tests remain green.
7. Relevant Go, frontend and E2E checks pass.
8. The architecture note and README describe implemented behavior, not future intent.

## Likely test locations

- backend/services/file/import_retry_test.go
- backend/services/file/upload_retry_test.go
- internal/app/import_retry_test.go
- frontend/src/modules/transfers.dom.test.ts
- frontend/src/ui/modals/failed-uploads-store.test.ts
- frontend/src/ui/notifications/activity-view.dom.test.ts
- frontend/e2e/folder-import.spec.ts
- frontend/e2e/upload-retry.spec.ts
- frontend/e2e/failed-upload-history.spec.ts

## Non-goals

- Claiming mobile folder retry works without native source ownership support.
- Changing unrelated transfer or storage subsystems.
