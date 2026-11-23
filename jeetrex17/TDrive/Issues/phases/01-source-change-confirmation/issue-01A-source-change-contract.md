# Issue 01A — Add an explicit backend contract for changed-source confirmation

## Goal

Extend the folder-import retry source checks from PR #200 so the backend can distinguish an ordinary safe retry from a source that changed and a source that the user explicitly approved to upload in its current form.

## Context

PR #200's ImportRetryFile records original source size and modification time. The retry path checks the path, opens the file, checks file identity and metadata, and returns source_changed when the original no longer matches.

That is safe, but it currently blocks the file. The feature request asks TDrive to warn the user and ask whether the current version may be uploaded.

## Scope

- Keep the original source fingerprint in the session-only retry record.
- Make source_changed a structured, stable outcome that the frontend can recognize.
- Add an explicit per-source confirmation field/contract for uploading the current version.
- When confirmation is present, capture the current file metadata at retry start and still validate the opened handle before sending.
- If the source disappears, is inaccessible, is not a regular file, or changes again between validation and open, return a blocked result.
- Preserve original ParentID, RelativePath, encryption and SkipExisting policy.
- Keep the retry source session-only.

## Acceptance criteria

1. A source matching its original metadata follows the existing safe retry path.
2. A changed source without explicit confirmation is not sent and returns source_changed.
3. A changed source with explicit confirmation may upload the current version only after a fresh backend check.
4. Missing/inaccessible sources cannot be approved into a successful send.
5. Confirmation is per source identity, not a blanket flag that silently approves every item in a batch.
6. Unknown remote outcomes and projection_pending remain non-retryable.
7. No full-file hash is introduced into the default path.
8. Tests prove no Telegram send occurs for an unconfirmed changed source.

## Likely files

- backend/services/file/import_retry.go
- backend/services/file/upload_single.go
- backend/services/file/upload_failures.go
- internal/app/import_retry.go
- generated Wails bindings
- frontend/src/types.ts

Re-read the live PR #200 branch before editing; do not duplicate code already merged upstream.

## Tests

- unchanged source accepted.
- changed size rejected without confirmation.
- changed modification time rejected without confirmation.
- confirmed changed source is sent using the current version.
- source replaced between validation and open is rejected/reported.
- missing source remains blocked.
- no send for outcome_unknown or projection_pending.
- original destination ParentID and SkipExisting policy are preserved.

## Non-goals

- Hashing every file.
- Persisting local paths or retry capability across application restarts.
- Mobile folder retry.
- Retrying temporary extracted archive members.
