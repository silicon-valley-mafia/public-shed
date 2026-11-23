# Issue 02B — Add Try Again to an individual failed file row

## Goal

Let the user retry exactly one eligible failed file from Activity → Transfers → selected batch → Failed.

## Depends on

- Phase 01 source-change confirmation.
- Issue 02A per-file retry state.
- PR #200 folder-import retry foundation.

## Scope

- Add a Try Again action to an eligible failed file row in ActivityDetails.svelte.
- Use the shared per-file selection function; do not call backend bindings directly from the Svelte component.
- For folder imports, use the original ImportRetryFile and its original ParentID.
- For flat-file uploads, reuse the existing RetryUploadToDriveFS path.
- Preserve drive ID, encryption and import conflict policy.
- Show queued/running/succeeded/failed-again/blocked state on the selected row.
- Disable the action while the source is queued or running.
- Keep the Review failed files dialog available for diagnostic detail.

## Acceptance criteria

1. Clicking Try Again for one row enqueues exactly one source.
2. Sibling files are not sent or modified.
3. Duplicate filenames do not cause the wrong file to be selected.
4. Successful files are not resent.
5. Non-retryable/unknown outcomes have no Try Again action.
6. Changed sources go through the Phase 01 confirmation.
7. Restored Activity history has no retry action.
8. Queue-full and wrong-drive errors are visible and do not consume eligibility.

## Likely files

- frontend/src/ui/notifications/ActivityDetails.svelte
- frontend/src/modules/transfers.ts
- frontend/src/ui/modals/failed-uploads-store.ts
- frontend/src/ui/notifications/activity-view.dom.test.ts
- frontend Playwright tests

## Non-goals

- Folder-level and batch-level actions.
- Persisting retry sources across restarts.
