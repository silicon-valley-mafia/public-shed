# Issue 04A — Add Try Again All to the selected Activity batch

## Goal

Expose a direct batch-level retry action in the Activity batch header.

## Depends on

- Phase 01 source-change confirmation.
- Phase 02 per-file retry state and selection helper.
- Phase 03 folder action should reuse the same selection path.
- PR #200 folder-import retry foundation.

## Scope

- Add Try Again All (N) to the selected batch header when eligible failures exist.
- N counts only eligible/confirmed failures, not every row with a Failed display status.
- Reuse the shared selection function with all eligible source IDs in this batch.
- Do not select successful, queued, running, canceled, unknown-outcome, projection-pending, resumable or blocked sources.
- Keep Review failed files for inspection.
- Scope the action to the selected Activity batch only. Do not sweep other batches, other drives or restored history.
- Preserve original drive, per-file destination, encryption and conflict policy.

## Acceptance criteria

1. The button appears only when at least one eligible failure exists.
2. Its count matches the actual selected sources.
3. Clicking it enqueues exactly the eligible failures in this batch.
4. Successful files are not resent.
5. Other Activity batches are unaffected.
6. Already queued/running items are not duplicated if an individual or folder retry is in progress.
7. Changed files pass through Phase 01 confirmation.
8. The action respects queue capacity and original-drive admission.

## Likely files

- frontend/src/ui/notifications/ActivityDetails.svelte
- frontend/src/modules/transfers.ts
- frontend/src/ui/modals/failed-uploads-store.ts
- Activity DOM and Playwright tests

## Non-goals

- A global Retry All across every Activity batch.
- Retrying files from restored history.
