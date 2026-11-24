# Phase 01 — Source-change confirmation

**User-visible capability:** TDrive detects that a failed file's local source changed after the original attempt and asks before uploading the current version.

The PR #200 draft already checks source size, modification time and file identity, but currently blocks a changed source instead of offering confirmation.

## Issues

1. [01A — Explicit backend confirmation contract](issue-01A-source-change-contract.md)
2. [01B — Confirm before uploading the changed version](issue-01B-confirm-changed-source.md)

## Completion criteria

- No changed file is sent before explicit approval.
- Approval applies only to the selected source(s).
- The backend revalidates after approval.
- Missing/inaccessible files remain blocked.
- No full-file hash is required for the default path; metadata limitations are documented.

Depends on the PR #200 folder-import retry foundation being accepted by the maintainer.
