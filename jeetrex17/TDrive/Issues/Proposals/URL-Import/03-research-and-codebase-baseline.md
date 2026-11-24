# Research and current TDrive codebase baseline

## Baseline and scope
Inspected 2026-11-24:
- Repository: https://github.com/jeetrex17/TDrive
- Branch: `master`
- Head: `fc01ebf357f1c52fa513db3a6f2a59befb10c2f2`
- PR #200 merge commit: same SHA.

This was a feature-focused inspection of upload service, failure classification, app APIs, transfer orchestration/queue, frontend wrappers, transfer architecture, updater downloader and queue E2E tests. It was not an exhaustive read of every repository file.

**Correction:** earlier discussion said PR #200 was an open draft. Live GitHub metadata on 2026-11-24 shows it is closed and merged into master. Recheck before implementation.

## Code findings

### backend/services/file/upload.go
The uploader uses seekable sources in key paths. The visible-source boundary accepts `io.ReadSeeker`, source path, filename and plaintext size. It validates size/destination, plans single versus multipart based on stored size (including encryption overhead), handles encryption, multipart and resumable paths and Telegram delivery.

**Implication:** a network `io.ReadCloser` is not a universal drop-in. Stage to a private seekable file, then reuse the existing uploader.

Source: https://github.com/jeetrex17/TDrive/blob/fc01ebf357f1c52fa513db3a6f2a59befb10c2f2/backend/services/file/upload.go

### backend/services/file/upload_failures.go
`UploadWithFailures` returns per-file failures keyed by original request index, not filename, and explicitly does not create a durable ordinary-upload retry journal. It distinguishes safe transient/not-sent failures from `outcome_unknown`, `projection_pending`, resumable uploads and cancellation.

**Implication:** acquisition failures need their own state. Upload retries must preserve the existing classifier. An error is not automatically permission for a fresh Telegram send.

Source: https://github.com/jeetrex17/TDrive/blob/fc01ebf357f1c52fa513db3a6f2a59befb10c2f2/backend/services/file/upload_failures.go

### internal/app/app.go
Current APIs include path-based `UploadToDriveFS(filePaths, parentIDs, encrypt)`, drive-bound retry and import methods. App paths stage some mobile sources, invoke the file service and release source paths. Drive-bound import rejects execution when the active channel differs from the captured expected channel.

**Implication:** the URL importer should stage, then call existing upload APIs with captured destination context.

Source: https://github.com/jeetrex17/TDrive/blob/fc01ebf357f1c52fa513db3a6f2a59befb10c2f2/internal/app/app.go

### frontend/src/modules/transfers.ts
Orchestrates upload/import, Activity creation, queue scheduling and runtime event reconciliation.

**Implication:** integrate URL transfer state into this orchestration or a focused helper module; do not create a second queue.

Source: https://github.com/jeetrex17/TDrive/blob/fc01ebf357f1c52fa513db3a6f2a59befb10c2f2/frontend/src/modules/transfers.ts

### frontend/src/modules/upload-queue.ts
Queue serializes batches, has one running entry and at most 10 waiting batches at inspected baseline. Waiting closures/source ownership are in-memory; release callbacks run when removed/completed/disposed. It uses drive policy when deciding runnable entries.

**Implication:** maintain temporary-source ownership through queue lifecycle and respect capacity and inactive-drive behavior.

Source: https://github.com/jeetrex17/TDrive/blob/fc01ebf357f1c52fa513db3a6f2a59befb10c2f2/frontend/src/modules/upload-queue.ts

### frontend/src/api/files.ts
Wraps generated Wails bindings and normalizes path-oriented upload/import APIs.

**Implication:** add typed bindings/results and regenerate/verify Wails bindings under project conventions.

Source: https://github.com/jeetrex17/TDrive/blob/fc01ebf357f1c52fa513db3a6f2a59befb10c2f2/frontend/src/api/files.ts

### docs/architecture/transfers-and-mounts.md
Describes distinct ordinary single, multipart, resumable and mounted-write publication/recovery boundaries. Documents Telegram acceptance uncertainty, automatic retry identity, shared upload semaphore, request-indexed failure outcomes and durable resumable jobs.

**Implication:** URL acquisition and Telegram upload need separate progress/retry state. Existing send outcome safeguards stay authoritative.

Source: https://github.com/jeetrex17/TDrive/blob/fc01ebf357f1c52fa513db3a6f2a59befb10c2f2/docs/architecture/transfers-and-mounts.md

### backend/updater/download.go
Contains a Go HTTP downloader specialized for trusted release assets. It requires expected SHA-256, validates size/checksum, writes .part and atomically renames. Not a general arbitrary URL downloader.

**Implication:** reuse safe patterns, not the code as a drop-in. Arbitrary URLs lack a trusted expected digest and need SSRF/redirect controls.

Source: https://github.com/jeetrex17/TDrive/blob/fc01ebf357f1c52fa513db3a6f2a59befb10c2f2/backend/updater/download.go

### frontend/e2e/upload-queue.spec.ts
Tests queue serialization, paused drive batches, queue removal/cancellation and source lifecycle. Extend these contracts for URL import.

Source: https://github.com/jeetrex17/TDrive/blob/fc01ebf357f1c52fa513db3a6f2a59befb10c2f2/frontend/e2e/upload-queue.spec.ts

## Baseline capability assessment

| Capability | Current baseline | Consequence |
|---|---|---|
| Local path upload | Existing | URL output becomes staged source |
| Seekable upload source | Existing | Stage where needed |
| Encryption/multipart/resumable | Existing | Reuse |
| Per-file upload failure | Existing, indexed | Preserve URL-to-path mapping |
| Queue/Activity | Existing | Extend, don't replace |
| Arbitrary URL resolution | Not found in inspected paths | New resolver |
| Platform extraction | Not found in inspected paths | New adapter |
| aria2c integration | Not found in inspected paths | Optional backend |
| FFmpeg URL import processing | Not established | Optional processor |
| Durable ordinary upload retry journal | Explicitly absent in UploadWithFailures | Do not assume |
| Durable acquisition journal | Not identified | Evaluate Phase 4 |

## External research

### yt-dlp
Upstream describes a feature-rich CLI downloader supporting many sites. Current README recommends FFmpeg/ffprobe for merging and post-processing and notes full YouTube support may require yt-dlp-ejs plus a supported JavaScript runtime. External downloader configuration exists. Verify exact versions and runtime at implementation.

https://github.com/yt-dlp/yt-dlp

### aria2
Official manual documents segmented HTTP/FTP downloads, multiple connections, multiple concurrent items and resuming partial HTTP(S)/FTP transfers where supported. These features are conditional; not every CDN supports resume or parallel segments.

https://aria2.github.io/manual/en/html/aria2c.html
https://github.com/aria2/aria2

### Cobalt
Cobalt API response patterns include tunnel, redirect, local-processing, picker and error. Hosted API documentation says hosted instances (including api.cobalt.tools) are not intended for use by other projects without permission. The repository defaults to AGPL-3.0; review license obligations before code reuse.

https://cobalt.tools/about/community
https://github.com/imputnet/cobalt
https://github.com/imputnet/cobalt/blob/main/docs/api.md
https://github.com/imputnet/cobalt/blob/main/LICENSE

### X API
Verify current official X API schema and an authorized real response for media variant fields. The example's `variants` property is not assumed universally available.

## Audit limitations
This inspection does not prove current account-specific X API entitlement, success for every platform extractor, final tool packaging on every OS, subprocess egress enforcement, or the final X resolver fallback order. Prototype and verify these before implementation.
