# Developer workflow: one issue at a time

## 1. Before starting

1. Read TDrive's current AGENTS.md, CONTRIBUTING.md, TESTING.md and docs/architecture/transfers-and-mounts.md.
2. Check current master, issue #186 and PR #200. The plan was researched against master base 52bc8de4727fb5804fca45a425cbc21c219ac59e and PR head 98014f0b992890900b728434158e2ff5694f0ded; these may have moved.
3. Confirm PR #200 is merged or agree with the maintainer how its draft work will be completed. Do not duplicate its folder-import retry implementation.
4. Read only the next issue file being implemented plus its referenced source/tests. Do not send the entire roadmap as one implementation task.
5. Inspect the developer's working tree and preserve unrelated changes.

## 2. One-issue execution loop

For each issue:

1. **Restate the contract.** Identify the user-visible behavior and explicit non-goals.
2. **Trace the current code.** Follow the call from Activity UI → transfer orchestration → queue → Wails binding → app method → file service. Read linked tests and architecture notes.
3. **Choose the smallest regression test first.** Reproduce the incorrect behavior before changing implementation where feasible.
4. **Implement only that issue.** Avoid unrelated refactors and avoid implementing later issue scopes early.
5. **Run focused tests.** Run the smallest Go/frontend/Playwright tests that protect the changed contract.
6. **Run broader checks.** Follow TDrive's AGENTS.md, TESTING.md and CI requirements.
7. **Review the diff.** Check generated Wails bindings, source path leakage, retry eligibility, index mapping, queue ownership, cancellation and error handling.
8. **Update documentation.** Update the relevant architecture note when implemented behavior changes.
9. **Report evidence.** List files changed, tests actually run, results, remaining risks and the next issue dependency.
10. **Stop.** Wait for review/approval before sending the next issue.

Do not ask the developer to implement all phases at once. Do not combine two issue files into one prompt unless the maintainer explicitly requests it.

## 3. Required invariants for every issue

- Only backend-marked safe failures can be retried.
- Never retry outcome_unknown or projection_pending as a fresh send.
- Never resend successful files.
- Preserve original drive, per-file ParentID, encryption and conflict policy.
- Use stable file identity/index, not filename.
- Keep source paths and retry actions in memory only.
- Keep retry selection within the selected batch/report.
- Do not bypass the existing upload queue, busy guard, semaphore or cancellation lifecycle.
- Do not allow duplicate clicks to enqueue the same source twice.
- Do not claim retry is available for restored Activity history, extracted archive members or mobile folder imports unless their source lifecycle is explicitly implemented.

## 4. Test strategy

### Backend

Use the existing file-service fixtures and fake Telegram client. Extend nearby tests rather than creating a second service harness.

Inject failures at the actual boundary:

- Before any send.
- Transient transport failure known not accepted.
- Remote acceptance followed by lost receipt.
- Outcome remains unknown.
- Local projection pending after remote acceptance.
- Source missing, permission denied, changed size, changed mtime, replaced path.
- Name conflict under SkipExisting.
- Cancellation while waiting and during a running window.

Assert actual sent bytes, destination ParentID, projected records, failure codes, retryable flags and absence of duplicate sends—not only mock call counts.

### App / Wails

Verify original drive pinning, transfer-busy rejection, cancellation, error mapping and generated bindings. A refused retry must not interrupt an existing upload.

### Frontend

Test the report state model independently from Svelte markup. Test subset selection, stable IDs, duplicate filenames, report updates, queued/running exclusion and repeat attempts.

### Activity / Playwright

Test one file, one folder and the selected batch. Include 100+ files, nested folders, multiple failed files in different folders, pagination, filters, changed sources, restored history and retry actions while another upload is active.

### Commands

Use the current repository's instructions; at the research baseline they specify:

- Go: go test . ./internal/... ./cmd/... ./backend/...
- Go vet: go vet . ./internal/... ./cmd/... ./backend/...
- Frontend: from frontend/, npm run typecheck
- Frontend lint: from frontend/, npm run lint
- Frontend tests: from frontend/, npm run test:coverage
- Frontend build: from frontend/, npm run build
- Browser contracts: from frontend/, npm run test:e2e

On Linux, the repository notes that direct Go build/test/vet commands need -tags=gtk3. Check current build guidance before running commands. Do not claim any command passed unless it was actually run.

## 5. Final review before each PR

- The diff contains only the current issue's scope.
- Generated Wails bindings match Go signatures.
- No local absolute paths, Telegram credentials, sessions or vault secrets enter persistent storage/logs.
- All retry results map back to the correct original file identity.
- Unknown remote outcomes cannot be retried.
- Successful files cannot be selected by any retry action.
- Documentation and tests describe the implemented behavior, not planned behavior.
- PR title/body follow CONTRIBUTING.md and reference issue #186 where appropriate.
