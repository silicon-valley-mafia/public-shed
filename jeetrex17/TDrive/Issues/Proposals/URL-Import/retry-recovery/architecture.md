# Target architecture: granular failed-upload retry

## 1. Scope

Add three retry granularities to the selected Activity upload batch:

1. One failed file.
2. Failed files in one containing folder.
3. All eligible failed files in the selected batch.

The first implementation targets desktop folder imports using the retry-source foundation in TDrive PR #200. Ordinary flat-file upload retry must keep working. Mobile folder retry and extracted-archive member retry are explicitly outside this first delivery.

## 2. Existing components to extend

| Responsibility | Existing location | Current role |
|---|---|---|
| Upload/import service | backend/services/file/import.go, import_tree.go | Walks folders and uploads files in bounded windows |
| Failure classification | backend/services/file/upload_failures.go | Indexes failures and decides whether a fresh send is safe |
| Folder retry source | backend/services/file/import_retry.go in PR #200 | Retains local source, original parent and source metadata for eligible failures |
| Wails integration | internal/app/import_retry.go in PR #200 | Pins retry to original drive and guards active transfer |
| API normalization | frontend/src/api/files.ts, frontend/src/types.ts | Normalizes UploadFailure and retry results |
| Queue | frontend/src/modules/upload-queue.ts | Serializes confirmed batches and owns queued source closures |
| Transfer orchestration | frontend/src/modules/transfers.ts | Creates reports, schedules retries and reconciles events |
| Activity model | frontend/src/ui/notifications/upload-activity-store.ts, activity-model.ts | Holds per-file display state and groups by containing folder |
| Activity detail UI | frontend/src/ui/notifications/ActivityDetails.svelte | Displays batch, file rows, folder headings and review action |
| Failure report | frontend/src/ui/modals/failed-uploads-store.ts, FailedUploadsModal.svelte | Holds session-only retry sources and the current bulk review dialog |
| Persisted display history | frontend/src/ui/notifications/failed-upload-history.ts | Persists safe display metadata, not source paths or retry actions |

Do not introduce a parallel upload engine. Reuse these components and existing backend upload APIs.

## 3. Retry eligibility is a safety contract

A file is eligible only when the backend explicitly reports it as retryable and the local source can be safely used.

| Outcome | Action |
|---|---|
| Temporary transport failure, known not accepted | Offer retry |
| Source unavailable due to a temporary permission/availability problem | Offer retry after the source becomes accessible |
| Source changed | Ask for explicit confirmation to upload the current version |
| Source missing | Block; ask user to restore/reselect it |
| outcome_unknown | Do not resend; reconcile/check the destination |
| projection_pending | Do not resend; allow synchronization/indexing to finish |
| resumable_upload | Use the existing resumable job controls |
| canceled | Do not treat as a retryable batch failure |
| empty_file | Explain that it cannot be uploaded |
| Destination/name conflict or unconfirmed remote state | Block and explain the required action |

Never derive eligibility from the display label Failed alone. Use the structured failure code and Retryable field.

## 4. Data model

### Stable file identity

A filename is not a unique identifier. Two different folders may contain the same filename. Keep a stable source identity for each item, tied to the original report/batch and original import index (or an opaque ID created at import time).

The current arrays have an important invariant: UploadFailure.Index indexes the original request, and folder-import retry sources are mapped to those same indexes. Preserve that mapping through every filter and subset operation. When a retry request creates a smaller array, retain a mapping back to the original source identity.

### Per-file attempt state

Replace whole-report consumption with per-file state. Suggested states:

- eligible — safe to retry.
- needs_confirmation — source changed; no send is allowed yet.
- queued — selected and admitted to the upload queue.
- running — retry is in progress.
- succeeded — retry completed successfully.
- failed_again — retry failed and the new outcome is known.
- blocked — missing source, uncertain outcome, conflict or another non-retryable result.

The exact enum names can follow TDrive's conventions. The required property is that state belongs to an individual source, not the entire report.

### Keep paths session-only

The retry payload may contain local paths while the current app session owns the source. Keep them in memory only. Do not add absolute paths, URI grants, retry tokens or retry closures to localStorage, notification persistence or saved Activity history.

The existing failed-upload history intentionally persists display-safe fields only. A restored Activity row after restart must remain non-retryable and explain that the user must reselect/re-import the source.

## 5. Source-change confirmation

### Capture

At the original folder import, retain:

- Local source path (session only).
- Resolved destination ParentID.
- Relative display path inside the imported folder.
- Original file size.
- Original modification time.
- Original SkipExisting policy.
- Stable source identity.

The PR #200 draft already captures size and modification time and checks the path/opened file.

### Validate

On a retry attempt:

1. Check the source path without following a symlink.
2. Require a regular file.
3. Open the file.
4. Stat the opened handle.
5. Check file identity with os.SameFile where supported.
6. Compare size and modification time.
7. Revalidate the opened file immediately before sending.

Use metadata checks by default; do not hash every file. Size + modification time + SameFile is a low-cost heuristic, not proof of byte-for-byte identity. A same-size edit with a restored timestamp can evade it, and SameFile semantics vary by OS.

### Confirm

If the source changed but still exists:

- Do not send it yet.
- Show the relative path/name and explain that it differs from the source used in the first attempt.
- Offer Upload current version and Skip.
- The confirmation applies only to the selected file(s).
- After confirmation, capture the current metadata as the new expected source and validate it again at open/send time.
- If the file changes again, return to confirmation rather than silently proceeding.

If the source is missing or inaccessible, confirmation cannot make it available. Keep it blocked.

## 6. Selection and destination preservation

Use one shared selection operation, conceptually:

retryFailedUploadSelection(reportId, sourceIds)

It should:

1. Resolve the active in-memory report.
2. Validate that the selected source IDs belong to that report.
3. Filter to safe/confirmed eligible files.
4. Reject IDs already queued or running.
5. Pin the original drive and original destination.
6. Preserve the original encryption choice.
7. For folder imports, preserve each source's own ParentID and SkipExisting policy.
8. Enqueue one retry batch through createUploadQueue.
9. Keep a mapping from retry results to original source IDs.
10. Update only the selected source states.

For ordinary flat-file uploads, use the existing RetryUploadToDriveFS path. For folder imports, use RetryImportedFilesForDrive with the per-file ImportRetryFile records. Do not flatten a nested import into one parent ID.

### Folder action semantics

Activity already groups imported files by their relative containing-folder path. A folder heading's Try Again action applies to eligible failed files whose immediate containing folder equals that heading's path.

Example:

- Photos/2024/January/a.jpg belongs to Photos/2024/January.
- Photos/2024/February/b.jpg belongs to Photos/2024/February.
- Retrying January does not include February or nested child folders.
- Each nested containing folder gets its own action.

This exact-parent rule is chosen to make the action predictable and align with the current groupFilesByFolder model. Do not silently implement recursive ancestor semantics.

### Batch action semantics

Try Again All applies to the selected Activity batch only. It does not retry other queued batches, other drives, unrelated reports or restored history. The existing Review failed files dialog remains available for diagnostics.

## 7. Activity UI

### Individual file

Add a Try Again action to a failed file row only when its session retry source exists and its outcome is eligible/confirmed. Disable it while queued/running. Keep the failure reason visible.

### Folder heading

Show Try Again (N) on a folder heading only when N eligible failures exist in that exact folder. Count only retryable failures, not all rows marked failed.

### Batch header

Show Try Again All (N) for the selected batch when it has eligible failures. Keep Review failed files for inspecting failure details. Do not make a single global button act on every batch in Activity.

### Restored history

When batch.restored is true or no session report exists, hide retry actions and explain that local sources are no longer retained. Do not resurrect absolute paths from storage.

### Pagination and filters

ActivityDetails pages rows and filters by status. Retry eligibility/counts must be derived from the full report, not only the currently rendered page or filtered rows. A folder action must still select all eligible files in that folder even when some of its rows are on another page.

## 8. Queue, event and concurrency behavior

- Use the existing upload queue; do not start a second backend upload concurrently.
- Respect its one-running-batch behavior and ten-waiting-batch limit.
- Preserve the original drive admission rule. If another drive is active, explain that the original drive must be opened.
- Use the existing transfer-busy guard and shared upload semaphore.
- Keep cancellation scoped to the retry batch and selected files.
- Keep picker-source ownership and release callbacks intact.
- Preserve the FlushEvents handshake. Upload results and Wails progress events can arrive separately; do not let late events update the next retry batch.
- Queue-full, drive-changed, source-missing and active-transfer errors must be visible and must not consume the source's retry eligibility.

## 9. Retry result reconciliation

Each retry attempt creates a new transfer/activity entry through the existing queue. Link it to the original batch/source identity so Activity can show the retry attempt without losing the first failure context.

On completion:

- Success: mark that source resolved; it must disappear from retryable counts.
- Known retryable failure: return that source to eligible/failed_again with the latest message.
- Source changed: needs_confirmation.
- Source missing/unavailable: blocked or eligible only according to the backend classification.
- Unknown remote outcome: blocked pending reconciliation; never auto-retry.
- Accepted but projection pending: show indexing/reconciliation state; never re-send.

Do not mark the entire report consumed when only one file or one folder was selected.

## 10. Platform and source boundaries

- Desktop folder import: target of PR #200 and this feature.
- Mobile folder import: keep current behavior until Android document-tree URI grants and native source ownership can be retained for retry. Do not pass stale temporary paths to the desktop retry API.
- Extracted archive members: do not retry their temporary extracted paths after cleanup. A future archive retry design would need to retain/reopen the original archive and deterministically rematerialize the selected entries.
- Large resumable uploads: continue to use their existing journal and Resume/Discard controls, not this ordinary failed-upload retry flow.

## 11. Acceptance criteria

The final user-visible behavior and safety rules are listed in [verification/acceptance-matrix.md](verification/acceptance-matrix.md).
