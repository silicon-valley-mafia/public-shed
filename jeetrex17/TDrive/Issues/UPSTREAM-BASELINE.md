# Upstream baseline

## Original request — issue #186

[Feature Request: Retry Failed Uploads Individually, by Folder, for the Entire Queue](https://github.com/jeetrex17/TDrive/issues/186)

The request describes a folder/batch with 100+ files and scattered failures. It asks for three actions under Activity → Transfers → selected queue → Failed:

- Retry one failed file.
- Retry all failed files in a selected folder.
- Retry all failed files in the selected queue.

It also requires reuse of the original source and destination, a source-existence/change check, confirmation before uploading a changed source, and exclusion of files that already uploaded successfully.

The maintainer clarified in the issue discussion that regular file uploads already have a bulk retry action in the Review failed files dialog, while folder uploads did not have that action. The reporter confirmed the affected workflow was a folder containing multiple subfolders.

## Existing work — PR #200

[Retry failed folder uploads from Activity](https://github.com/jeetrex17/TDrive/pull/200)

At research time:

- State: open
- Draft: yes
- Base: master at 52bc8de4727fb5804fca45a425cbc21c219ac59e
- Head: fix/folder-upload-retry at 98014f0b992890900b728434158e2ff5694f0ded
- Scope in PR description: same-session desktop folder-import retry, original subfolder destinations and source validation.

The draft adds a folder-import retry source record, captures eligible failures from import upload windows, and retries those sources using their original resolved parent IDs. It excludes temporary extracted-archive copies and refuses to resend unknown remote outcomes. It adds backend, app, frontend and E2E coverage.

The draft does **not** implement the full three-level interaction requested in #186. In particular, the Activity file rows and folder headings still do not expose individual or folder-scoped Try Again actions; the new folder-import path is reviewed through the existing failed-upload dialog and its bulk retry button. Changed sources are rejected as non-retryable; the requested explicit confirmation to upload the current version is not present.

## Required prerequisite

Before implementing this plan:

1. Re-check whether PR #200 is still open, merged, superseded or changed.
2. Read its current diff and review comments.
3. If it is merged, use the merged implementation as the baseline.
4. If it remains a draft, do not duplicate its changes. Ask the maintainer to finish/review it, or agree on a coordinated update to that branch.
5. Rebase all issue work on the current master after the baseline is settled.

Do not create a second implementation of the same folder-import retry foundation.
