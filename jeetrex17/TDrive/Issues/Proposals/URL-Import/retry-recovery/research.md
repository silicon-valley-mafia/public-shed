# Research: TDrive failed-upload recovery

Research date: 2026-11-23

## 1. Repository inventory and audit scope

The public repository is [jeetrex17/TDrive](https://github.com/jeetrex17/TDrive). Its default branch is master. The recursive Git tree was retrieved successfully (not truncated) and contained:

- 1,659 tracked files
- 68,878,785 bytes across those tracked files
- 770 Go files
- 529 TypeScript files
- 94 Svelte files
- 27 Markdown files
- Binary/media assets and platform-specific native projects make up the remainder.

The tree inventory is repository-wide. The source review was focused on the feature's complete known path across backend import/upload, Wails app bindings, frontend transfer queue, Activity, failed-upload reporting, persistence boundaries, tests and architecture documentation. It was **not** a literal reading of all 1,659 files. Unrelated subsystems such as media playback, gallery rendering, mount implementation, photo backup internals and platform packaging were not exhaustively audited. Do not describe this as a full-repository code review.

## 2. User request and maintainer clarification

The authoritative request is [issue #186](https://github.com/jeetrex17/TDrive/issues/186).

The motivating case is a folder upload containing many files and nested directories, where a small number fail sporadically. The user should not have to locate those local files, recreate the destination path and upload them manually.

The issue requests three granularities:

- Individual file.
- One folder containing failed files.
- Entire selected upload queue/batch.

It also requires original destination preservation, source existence/change checks, confirmation if the source changed, and never resending successful files.

The issue discussion narrows the current gap: the maintainer says regular file uploads already support bulk retry through Review failed files; the reporter confirms the missing action was encountered while uploading folders with multiple subfolders.

## 3. Existing implementation at the PR #200 base

The PR base is master commit 52bc8de4727fb5804fca45a425cbc21c219ac59e. Re-check this before coding because upstream may move.

### Backend upload failure contract

Relevant files:

- backend/services/file/upload_failures.go
- backend/services/file/upload.go
- backend/services/file/upload_single.go
- backend/services/file/import.go
- backend/services/file/import_tree.go
- backend/services/file/upload_retry_test.go
- docs/architecture/transfers-and-mounts.md

UploadFailure identifies a failed input by its request index, not by filename. This matters because different folders can contain duplicate filenames.

The Retryable flag is the safety gate. A visible status of Failed alone is not permission to send again. Current classifications distinguish safe transient transport/source failures from outcomes that need reconciliation or a different recovery control.

Important non-retry cases include:

- outcome_unknown: Telegram may have accepted the send but the receipt was lost.
- projection_pending: Telegram accepted the file; local indexing is pending.
- resumable_upload: use the resumable job's controls.
- canceled: user cancellation is not an ordinary failed upload.
- empty_file: the file is not uploadable.

The architecture note explains the key invariant: an error does not prove that Telegram did not accept a file. Never create a new send when acceptance is uncertain.

### Regular file uploads

The existing frontend flow in frontend/src/modules/transfers.ts collects UploadFailure results, keeps the original selected paths and destination in an in-memory FailedUploadsPayload, and opens FailedUploadsModal.svelte. retryFailedUploads selects only failures marked retryable and schedules them through the existing upload queue.

This is the bulk retry path the maintainer referred to in issue #186. It is not the requested per-file and per-folder action set.

### Folder imports

Folder imports are walked and uploaded in bounded windows. The current import machinery retains paths and resolved destination parent IDs while each window runs, but the ordinary import result does not create a durable retry journal.

PR #200 adds:

- ImportRetryFile with the local source path, resolved ParentID, display RelativePath, original Size and ModifiedAt, failure details and the original SkipExisting policy.
- RunImportWithRetry to collect only safely retryable original local sources.
- RetryImportedFiles to reuse the existing upload windows and each file's original parent ID without recreating folders.
- An app-level method pinned to the original drive and protected by the existing transfer-busy guard.
- Frontend integration that carries the retry sources in session memory and uses the existing Activity failed-upload review dialog.

The PR excludes extracted archive contents because their extracted paths live in a temporary directory that is removed after import. It also keeps folder-import retry desktop-only; mobile picker sources have different ownership and URI-grant lifecycles.

### Source identity check in PR #200

The draft captures file size and modification time. Before a retry, the backend checks that the path still exists, is a regular file, and matches the captured metadata. It then opens the file and checks the opened file against the earlier path check using os.SameFile and the metadata again.

This is a useful low-cost guard and avoids hashing every file. It is not cryptographic proof that the content is unchanged. Go documents that SameFile is platform-dependent: Unix commonly compares device/inode identity, while other systems may base the result on path names. Filesystems can also round or truncate modification times. A same-size edit with a restored timestamp may evade metadata checks.

The draft returns a source_changed failure instead of offering a confirmation flow. The requested behavior is to warn the user and ask whether the current version may be uploaded. The backend must still revalidate after confirmation; confirmation must not become an unconditional bypass of source checks.

## 4. Activity and queue architecture

Relevant files:

- frontend/src/modules/upload-queue.ts
- frontend/src/modules/transfers.ts
- frontend/src/ui/notifications/upload-activity-store.ts
- frontend/src/ui/notifications/activity-model.ts
- frontend/src/ui/notifications/ActivityDetails.svelte
- frontend/src/ui/modals/failed-uploads-store.ts
- frontend/src/ui/modals/FailedUploadsModal.svelte
- frontend/src/ui/notifications/failed-upload-history.ts
- frontend/src/ui/notifications/FailedUploadReports.svelte
- frontend/src/api/files.ts
- frontend/src/types.ts

Findings:

1. ActivityDetails already renders per-file status rows and groups imported files by their relative containing-folder path through groupFilesByFolder.
2. Folder headings already display counts, including failed counts, but currently only expand/collapse.
3. Failed file rows display the reason and status but do not have a Try Again action.
4. The batch header currently offers Review failed files when a session report is available.
5. FailedUploadsModal has a bulk Retry failed files button.
6. retryFailedUploads currently consumes an entire FailedUploadsPayload through a WeakSet. This is sufficient for one bulk action, but not for repeated partial actions on individual files or folders.
7. The upload queue serializes interactive batches, has one running batch and at most ten waiting batches, and keeps queued closures/source ownership in memory.
8. Activity display history can be restored, but source paths and retry actions are deliberately not persisted. The saved failed-upload history is display-only and redacts local locations.
9. Wails calls and runtime events can arrive separately. transfers.ts uses withEventsDelivered/FlushEvents so a batch's late progress events are flushed before the next queued batch reuses event identifiers.

## 5. Architectural constraints that the feature must preserve

- Use stable file identity/request indexes; never key retry selection by filename alone.
- Pin retries to the original drive and destination.
- Folder-import retries must retain each file's resolved ParentID, not flatten all files to one parent.
- Preserve the original encryption choice and Skip/Keep Both import policy.
- Only retry failures explicitly marked safe.
- Do not retry uncertain sends or accepted-but-unprojected files.
- Do not resend successful files.
- Keep local absolute paths and retry capabilities session-only. Do not add them to localStorage or persisted Activity history.
- Respect the existing single-upload admission, queue limit, upload semaphore, cancellation and picker-source ownership.
- Keep mobile folder-import retry out of scope until native source/URI lifetime can be retained safely.
- Do not promise retry of extracted archive members while their extracted source paths are temporary.

## 6. External technical research

### Telegram send deduplication

Telegram's messages.sendMedia method accepts random_id to avoid resending the same message. Telegram's updates documentation explains that random IDs are used for server-side deduplication and that updateMessageID maps the random ID to the resulting message ID.

This supports the existing distinction between automatic transport retry of the same send operation and a user-initiated new upload. A new manual retry is a new operation; it is safe only when the previous operation is known not to have been accepted. Do not weaken the outcome_unknown guard.

- https://core.telegram.org/method/messages.sendMedia
- https://core.telegram.org/api/updates

### Go source identity metadata

Go's os package documents os.SameFile as platform-dependent. Its Chtimes documentation notes that the underlying filesystem may truncate or round modification times. Size + modification time + SameFile is therefore a practical low-cost change detector, not a content hash or absolute identity guarantee.

- https://pkg.go.dev/os#SameFile
- https://pkg.go.dev/os#Chtimes

### Wails event delivery

Wails describes its event system as asynchronous; Emit does not wait for handlers. TDrive's own FlushEvents handshake is therefore an important application-level ordering contract and should be retained when adding retry batches.

- https://v3.wails.io/features/events/system/

## 7. Gaps confirmed

| Requirement | At master / existing design | PR #200 draft | Full target |
|---|---|---|---|
| Ordinary file bulk retry | Available in review dialog | Unchanged | Preserve |
| Folder-import bulk retry | Missing | Adds desktop same-session retry | Preserve and expose clearly |
| Original nested destination | Import knows per-file parent during first attempt | Retains per-file ParentID | Preserve for every retry granularity |
| Individual retry | Missing | Missing | Required |
| Folder-scoped retry | Missing | Missing | Required |
| Retry all in selected batch | Existing bulk dialog for regular files | Folder import gains bulk dialog retry | Add direct Activity batch action |
| Changed source confirmation | Not available | Changed source is blocked | Required |
| Repeated partial retry state | Whole report is consumed by WeakSet | Still whole-report consumption | Per-file attempt state required |
| Restart retry | Intentionally unavailable | Intentionally session-only | Keep session-only |

## 8. Risks and decisions

- The UI phrase “entire queue” must mean the selected Activity batch, not every failed item across unrelated historical batches. This plan uses selected-batch scope.
- A folder action applies to failed files whose immediate containing folder equals that folder heading. Nested folders have their own headings/actions. It does not implicitly include descendants.
- Changed-source confirmation authorizes the current version of that one source, not the original bytes. The backend must re-check the file immediately before opening/sending.
- Metadata-only checking can miss content changes if size and timestamps are preserved. Do not claim otherwise. Full hashing is excluded from the default path.
- If a file is missing, inaccessible or still changing after confirmation, keep it blocked and explain the action needed.
- A retry that returns outcome_unknown must move to reconciliation/attention, not remain automatically retryable.
- The PR #200 branch is draft work and may change. Its exact implementation must be re-read before each issue is handed to a developer.
