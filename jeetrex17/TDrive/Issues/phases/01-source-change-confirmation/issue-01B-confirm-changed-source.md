# Issue 01B — Ask before uploading a changed source

## Goal

When a retry discovers that a failed file's local source changed, show a clear confirmation before uploading the current version.

## Depends on

- Issue 01A: backend confirmation contract.
- PR #200 folder-import retry foundation.

## Scope

- Recognize the structured source_changed result.
- Show the affected relative path/name and explain that the local file differs from the version used in the first attempt.
- Offer explicit actions:
  - Upload current version.
  - Skip this file.
- For a multi-file retry, present the changed files together in one review surface so the user can approve only the listed/selected files instead of receiving an interrupting dialog for every file.
- Send confirmation only for the exact stable source IDs approved by the user.
- If the user skips or closes the prompt, do not send those files.
- If a source changes again after confirmation, rely on the backend recheck and return it to confirmation rather than silently uploading.
- Keep the existing failure details and original destination visible.

## Acceptance criteria

1. A changed source is never uploaded automatically.
2. The confirmation clearly distinguishes uploading the current version from retrying the original version.
3. User approval is scoped to selected source IDs.
4. Cancel/Skip produces no Telegram send.
5. Missing files do not show an Upload current version action.
6. The UI shows a useful result if the source changes again before send.
7. The action is keyboard accessible and works at desktop window sizes.
8. Local absolute paths are not persisted.

## Likely files

- frontend/src/modules/transfers.ts
- frontend/src/ui/modals/FailedUploadsModal.svelte or a focused confirmation component
- frontend/src/ui/modals/failed-uploads-store.ts
- frontend/src/types.ts
- frontend DOM and Playwright tests

## Non-goals

- A general file picker/reselect workflow.
- Full-file hashing.
- Mobile folder-import retry.
