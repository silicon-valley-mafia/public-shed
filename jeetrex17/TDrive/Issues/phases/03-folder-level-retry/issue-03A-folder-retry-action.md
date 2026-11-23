# Issue 03A — Add Try Again to a folder containing failed files

## Goal

Add a folder-scoped retry action to each folder heading in the Activity batch detail.

## Depends on

- Phase 01 source-change confirmation.
- Phase 02 per-file retry state and individual retry selection.
- PR #200 folder-import retry foundation.

## Scope

- Extend the existing folder heading in ActivityDetails.svelte with Try Again (N) when that folder contains N eligible failures.
- Use the existing relative-path grouping in activity-model.ts.
- Apply the action to failed files whose immediate containing folder equals the heading path.
- Nested child folders remain separate groups; the parent action does not implicitly include descendants.
- Select only backend-marked retryable/confirmed sources.
- Reuse the same selection function as individual retry.
- Preserve every selected file's original ParentID, source identity, encryption and SkipExisting policy.
- Compute counts from the full report, not only the current page or active filter.

## Acceptance criteria

1. A folder with no eligible failures has no retry action.
2. A folder with eligible failures shows the correct count.
3. Clicking it retries only eligible failed files in that exact containing folder.
4. Successful files and sibling/nested folder files are untouched.
5. Duplicate filenames in different folders are handled by stable identity.
6. Folder collapse, pagination and file filters do not change the selected set.
7. Changed files require Phase 01 confirmation.
8. Queue and drive admission rules are preserved.

## Likely files

- frontend/src/ui/notifications/ActivityDetails.svelte
- frontend/src/ui/notifications/activity-model.ts
- frontend/src/modules/transfers.ts
- frontend DOM and Playwright tests

## Non-goals

- Recursive parent-folder retry.
- Retrying all files in the batch.
- Changing how the import walker creates folders.
