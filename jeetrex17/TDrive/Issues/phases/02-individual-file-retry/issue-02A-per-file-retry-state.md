# Issue 02A — Track retry state per failed source

## Goal

Replace whole-report retry consumption with per-file retry state so a user can retry one item, then another item, without losing the remaining retry actions.

## Context

The current retry flow uses a WeakSet of consumed FailedUploadsPayload objects. That prevents a report from being bulk-retried twice, but it consumes the whole report even when the user selects only one file. Granular actions need state per source.

## Scope

- Give each failed source a stable identity within its original Activity batch/report. Do not use filename as identity.
- Preserve the mapping between original UploadFailure.Index, paths and ImportRetryFile records.
- Track per-source state: eligible, needs_confirmation, queued, running, succeeded, failed_again or blocked (names may follow existing conventions).
- Add one shared selection function that accepts a report/batch ID and an explicit list of source IDs.
- Reject source IDs not belonging to the report.
- Reject IDs already queued or running.
- Update only selected source states.
- Keep source paths and retry state in memory only.
- Keep failed-upload-history persistence display-only.

## Acceptance criteria

1. Duplicate filenames in separate folders remain independently selectable.
2. Selecting one source does not consume or hide other eligible sources.
3. The same source cannot be admitted twice concurrently.
4. A retry result maps back to the correct original source even after the retry request array is compacted.
5. Success removes that source from eligible counts.
6. Safe failure can become eligible again after its latest result is known.
7. Unknown outcome becomes blocked, not eligible.
8. No absolute source path is added to localStorage or persisted Activity history.

## Likely files

- frontend/src/ui/modals/failed-uploads-store.ts
- frontend/src/modules/transfers.ts
- frontend/src/types.ts
- frontend/src/ui/notifications/upload-activity-store.ts
- related unit/DOM tests

## Non-goals

- Adding the visible Try Again buttons (issue 02B).
- Folder or batch selection UI.
- Changing backend retry eligibility rules.
