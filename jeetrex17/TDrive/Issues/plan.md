# Feature plan: granular failed-upload retry

## Objective

Complete the feature requested in [issue #186](https://github.com/jeetrex17/TDrive/issues/186): retry one failed file, failed files in one folder, or all eligible failures in the selected upload batch directly from Activity.

The target is a nested folder upload with scattered failures. The user should not have to find the local files again or rebuild their remote folder destinations.

## Product behavior

From Activity → Transfers → selected upload batch:

- A failed eligible file has its own Try Again action.
- A folder heading with eligible failures has a Try Again (N) action.
- The batch header has Try Again All (N), where N is the number of eligible failures in that batch.
- Successful files are never sent again.
- Files with uncertain remote acceptance are never blindly resent.
- The original drive, destination parent for each file, encryption setting and import conflict policy are preserved.
- If a source changed, the user is warned and must explicitly approve uploading the current version.
- Missing sources and non-retryable outcomes remain blocked with an actionable explanation.
- Restored Activity history after app restart remains display-only; source paths are not persisted.

## Upstream prerequisite — not a new implementation issue

Settle [PR #200](https://github.com/jeetrex17/TDrive/pull/200) first. It is an open draft at the research baseline and already implements the desktop folder-import retry foundation. Re-read the live PR and current master before starting. Do not duplicate it.

The work below assumes the folder-import retry source records and RetryImportedFiles backend path from #200 are merged or otherwise accepted by the maintainer.

## Phases and issue sequence

Phases are user-visible feature additions. Issues inside them are intentionally small and should be handed to the developer one at a time.

| Order | Phase | Issue | Deliverable | Depends on |
|---:|---|---|---|---|
| 1 | 01 — Source-change confirmation | 01A | Backend contract for checking and explicitly allowing a changed source | PR #200 baseline |
| 2 | 01 — Source-change confirmation | 01B | User confirmation before uploading the current changed version | 01A |
| 3 | 02 — Individual file retry | 02A | Stable per-file retry identity and attempt state | PR #200 baseline |
| 4 | 02 — Individual file retry | 02B | Try Again action on one eligible failed file row | 01A, 01B, 02A |
| 5 | 03 — Folder-level retry | 03A | Try Again (N) on each containing-folder heading | 02A, 02B |
| 6 | 04 — Batch-level retry | 04A | Try Again All (N) on the selected Activity batch | 02A, 02B |
| 7 | 05 — Retry-state safety | 05A | Partial/repeated retry state, duplicate-click protection and accurate outcomes | 02B, 03A, 04A |
| 8 | 06 — Integration verification | 06A | End-to-end and regression coverage for the complete behavior | All previous issues |

### Phase 01 — Source-change confirmation

A source mismatch must not silently upload new bytes. The current PR #200 draft detects a mismatch and blocks it. Extend that into an explicit, per-file confirmation path. Missing or inaccessible files are not confirmable.

### Phase 02 — Individual file retry

The user can retry one failed file without retrying its siblings. This establishes the smallest retry unit and the state model reused by folder and batch actions.

### Phase 03 — Folder-level retry

The user can retry eligible failures belonging to one displayed containing-folder group. The action must not include successful files or files in sibling/nested folder groups.

### Phase 04 — Batch-level retry

The user can retry all eligible failures in the selected Activity batch with one action. It must not sweep other batches or restored history.

### Phase 05 — Retry-state safety

The original report must remain accurate while retries are queued, running, successful, failed again or blocked. A retry action must not consume the entire report when only one subset was selected.

### Phase 06 — Integration verification

Verify the complete workflow across backend, Wails binding, frontend Activity, queue behavior, privacy boundaries and supported platforms.

## Issue files

- Phase 01:
  - [01A — Backend source-change confirmation contract](phases/01-source-change-confirmation/issue-01A-source-change-contract.md)
  - [01B — Confirm before uploading a changed source](phases/01-source-change-confirmation/issue-01B-confirm-changed-source.md)
- Phase 02:
  - [02A — Per-file retry state](phases/02-individual-file-retry/issue-02A-per-file-retry-state.md)
  - [02B — Individual Try Again action](phases/02-individual-file-retry/issue-02B-individual-retry-action.md)
- Phase 03:
  - [03A — Folder-level Try Again action](phases/03-folder-level-retry/issue-03A-folder-retry-action.md)
- Phase 04:
  - [04A — Batch-level Try Again All action](phases/04-batch-level-retry/issue-04A-batch-retry-all.md)
- Phase 05:
  - [05A — Partial and repeated retry safety](phases/05-retry-state-safety/issue-05A-repeat-retry-safety.md)
- Phase 06:
  - [06A — End-to-end verification](phases/06-integration-verification/issue-06A-e2e-regression.md)

## Definition of done

The feature is complete when:

1. A 100+ file nested folder upload with scattered failures can be recovered without manually reselecting files.
2. Individual, folder and selected-batch actions each retry only their intended eligible files.
3. Original nested destinations and encryption/import policies are preserved.
4. Changed files require explicit confirmation; missing files cannot be force-retried.
5. Successful files and uncertain sends are never blindly resent.
6. Partial and repeated attempts remain correctly represented in Activity.
7. Local source paths and retry capabilities are not written to persistent Activity history.
8. The focused and full test suites pass, and desktop/mobile boundaries are documented accurately.
