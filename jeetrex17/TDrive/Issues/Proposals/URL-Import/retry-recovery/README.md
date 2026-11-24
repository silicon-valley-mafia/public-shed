# TDrive — Failed Upload Recovery

This directory contains the researched feature plan for improving failed-upload recovery in [jeetrex17/TDrive](https://github.com/jeetrex17/TDrive).

## User goal

From Activity → Transfers → a selected upload batch, let the user retry:

1. One failed file.
2. All eligible failed files in one folder.
3. All eligible failed files in the selected upload batch.

The retry must preserve the original source, destination folder, drive, encryption choice and folder structure. Successful files must never be uploaded again.

## Start here

- [Plan and issue sequence](plan.md)
- [Repository and upstream research](research.md)
- [Target architecture](architecture.md)
- [Developer workflow](workflow.md)
- [Upstream baseline](UPSTREAM-BASELINE.md)
- [Verification matrix](verification/acceptance-matrix.md)

## Feature phases

| Phase | Deliverable |
|---|---|
| 01 — Source-change confirmation | Detect changed/missing local sources and ask before uploading a changed version |
| 02 — Individual file retry | Retry one eligible failed file from its Activity row |
| 03 — Folder-level retry | Retry eligible failures in one containing folder |
| 04 — Batch-level retry | Retry all eligible failures in the selected batch |
| 05 — Retry-state safety | Support partial/repeated attempts without duplicate submissions or stale state |
| 06 — Integration verification | Exercise the complete workflow and protect existing upload behavior |

Each phase adds a user-visible capability. The phases have dependencies, but they are not generic project-status stages.

## Important upstream context

- Original request: [TDrive issue #186](https://github.com/jeetrex17/TDrive/issues/186)
- Existing implementation work: [TDrive draft PR #200](https://github.com/jeetrex17/TDrive/pull/200)

PR #200 is the baseline to review, not work to duplicate. At the time this plan was researched, it was an open draft. Re-check its status and current master before implementation.

## One-issue-at-a-time rule

The issue drafts under each phase are deliberately small. Give the developer one issue file at a time, wait for its implementation and review, verify the result, and only then provide the next issue. Do not send the entire roadmap as one implementation prompt.

## Audit qualification

The repository tree was inventoried and the upload/retry/Activity subsystem, its linked architecture notes, tests and contributor instructions were inspected. This is a feature-focused audit, **not a claim that every one of the repository's 1,659 tracked files or binary assets was read**. See [research.md](research.md) for the exact scope and remaining limitations.
