# Acceptance and verification matrix

Use this matrix during implementation and before declaring issue #186 complete. Tests must assert consequences at the relevant boundary.

## User-visible behavior

| Scenario | Expected behavior |
|---|---|
| One retryable failed file | Its row offers Try Again; only that file is queued |
| Folder with retryable failed files | Its heading offers Try Again (N); only eligible failures in that exact containing folder are queued |
| Selected batch has retryable failures | Batch header offers Try Again All (N); only that batch's eligible failures are queued |
| Successful file in the same batch | Never selected or sent again |
| Two files share a filename in different folders | Actions select the correct source by stable identity/index |
| Nested child folder | Has its own containing-folder action; parent folder action does not silently include descendants |
| Retry action clicked twice quickly | One retry admission only |
| Retry action while source is already queued/running | Disabled or rejected without duplicate send |
| Retry queue is full | Clear error; source remains eligible |
| Original drive is not active | Clear instruction to open the original drive; no send |
| Original destination folder no longer exists | Clear blocked failure; do not silently upload elsewhere |
| Retry succeeds | Source becomes resolved and is removed from eligible counts |
| Retry fails safely again | Latest failure is shown and another deliberate attempt remains possible |
| Retry outcome becomes unknown | Source is blocked pending reconciliation; no fresh send |
| Activity restored after restart | Display only; no retry action because source path was not persisted |
| Activity history cleared | Does not cancel active work or delete backend recovery journals |

## Source validation

| Source condition | Expected behavior |
|---|---|
| Same path, same size and mtime, same file | Eligible, subject to backend's final open/stat check |
| File size changed | Show changed-source confirmation; do not send before approval |
| Modification time changed | Show changed-source confirmation; do not send before approval |
| File replaced at same path | Detect through available file identity/metadata checks; confirm or block |
| File missing | Block; ask user to restore/reselect |
| File inaccessible | Explain permissions/availability; do not claim it was uploaded |
| File changes again after confirmation | Revalidate and ask again or block; never silently upload an unconfirmed version |
| Same-size content edit with restored timestamp | Document metadata heuristic limitation; no claim of byte identity without hashing |

## Remote outcome safety

| Backend result | Retry behavior |
|---|---|
| temporary_failure / known not sent | Eligible |
| source_unavailable | Eligible only after source becomes accessible and backend confirms safe |
| source_changed | Needs explicit confirmation |
| source_missing | Blocked |
| outcome_unknown | Not retryable; reconcile/check Telegram state |
| projection_pending | Not retryable; wait for sync/indexing |
| resumable_upload | Use Resume/Discard on the resumable job |
| canceled | Not included in retryable failure count |
| empty_file | Not retryable |

## Folder import preservation

- Original drive ID is preserved.
- Each failed file retains its original resolved ParentID.
- RelativePath is used for display/grouping only; it is not used as a local filesystem path.
- Original encryption choice is preserved.
- Original SkipExisting/Keep Both behavior is preserved.
- A retry does not recreate the folder tree or re-upload successful siblings.
- Import windows remain bounded (128 files in the current implementation) and the overall import item limit remains enforced.

## Privacy and lifecycle

- Absolute source paths remain session-only.
- Persisted failed-upload history contains only display-safe information.
- No Telegram API hash, session, vault password, private key or local database is written to this public repository.
- Logout/teardown releases session sources according to existing ownership rules.
- Mobile folder import remains unchanged until native URI/tree source ownership is explicitly supported.
- Extracted archive temporary files are not presented as retryable after cleanup.

## Regression suite

### Backend

- Existing upload retry tests.
- Folder-import retry source tests from PR #200.
- Tests for known-not-sent vs accepted-but-unknown vs projection-pending outcomes.
- Tests for source changed/missing/replaced, original ParentID, SkipExisting, cancellation and duplicate-name indexes.

### Frontend and Activity

- FailedUploadsPayload/report store tests.
- upload-activity-store and activity-model tests.
- transfers DOM tests.
- activity-view DOM tests.
- folder-import and upload-retry Playwright tests.
- failed-upload-history Playwright tests.

### Platform checks

- Desktop Windows, macOS and Linux path/metadata behavior.
- Mobile regression: no stale path retry and no loss of native picker ownership.
- Verify app restart/logout behavior and that restored reports are read-only.

## Release gate

Do not mark the feature complete until:

1. All issue-level tests pass.
2. The full relevant Go and frontend suites pass.
3. Playwright verifies the three retry granularities end-to-end.
4. No regression allows duplicate sends after uncertain acceptance.
5. The final diff and documentation have been reviewed against this matrix.
