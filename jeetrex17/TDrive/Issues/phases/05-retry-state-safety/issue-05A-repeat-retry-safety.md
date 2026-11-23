# Issue 05A — Make partial and repeated retry attempts state-safe

## Goal

Ensure Activity remains correct when a user retries one file, then a folder, then the remaining batch—or retries a file that fails again.

## Depends on

- Issues 02A and 02B.
- Issues 03A and 04A.

## Scope

- Update original source state only for the files included in an attempt.
- Keep other eligible failures available.
- Prevent the same source from being selected while queued or running.
- On success, mark the source resolved and remove it from retryable counts.
- On a known retryable failure, show the latest reason and allow a later deliberate retry.
- On source_changed, require confirmation again.
- On outcome_unknown or projection_pending, block fresh retry and direct the user to reconciliation/sync.
- Link retry Activity entries to the original batch/source identity (or implement an equally clear relationship).
- Preserve cancellation and queue-removal semantics.
- Ensure a retry result from a prior attempt cannot overwrite a newer attempt's state.

## Acceptance criteria

1. Individual retry followed by Retry All excludes the already queued/running item.
2. Folder retry followed by Retry All excludes items already in that attempt.
3. A retry that succeeds is never offered again.
4. A retry that safely fails again remains eligible with its latest error.
5. A retry with uncertain acceptance is blocked.
6. Out-of-order/late events cannot mutate a later retry attempt.
7. Clearing completed Activity history does not cancel active retries or delete backend recovery journals.
8. Logout/teardown releases only sources owned by the session.

## Likely files

- frontend/src/modules/transfers.ts
- frontend/src/modules/upload-queue.ts
- frontend/src/ui/modals/failed-uploads-store.ts
- frontend/src/ui/notifications/upload-activity-store.ts
- frontend/src/ui/notifications/activity-model.ts
- related tests

## Non-goals

- Persisting retry sources across application restart.
- Adding a second upload scheduler.
- Weakening backend remote-outcome safety.
