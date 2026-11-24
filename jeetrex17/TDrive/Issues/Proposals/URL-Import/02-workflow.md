# End-to-end user and system workflows

## Direct single-file URL
1. User selects Import from URL and pastes one URL.
2. UI gives basic syntax feedback; backend revalidates every security-sensitive property.
3. Intake rejects unsupported schemes and assigns a transfer/item ID.
4. Direct resolver safely probes/fetches. HEAD is only an optimization because some servers do not implement it correctly.
5. Resolver returns filename, type, size if available, approved final URL and warnings.
6. UI displays review and captures destination drive/folder and encryption.
7. User confirms; transfer enters the existing queue.
8. Go HTTP streams into private staging with bounded memory and byte/time limits.
9. Staged file is validated and passed to existing TDrive upload API.
10. Existing uploader handles encryption, size planning, multipart/resumable logic, Telegram publication and local projection.
11. Activity reports acquisition and upload separately.
12. Staging is cleaned or retained according to safe retry policy.

## Batch direct URLs
1. User pastes multiple URLs, one per line.
2. Intake assigns a distinct stable ID to each URL.
3. Each is validated/resolved independently with bounded concurrency.
4. UI shows valid, invalid, unsupported, unsafe, inaccessible and metadata-incomplete entries.
5. User selects entries and confirms.
6. Destination/encryption context is captured.
7. Selected items enter the existing TDrive queue; inspected baseline allowed at most 10 waiting batches.
8. Each URL acquires/uploads independently. One item failure does not abort unrelated items.
9. Per-item and whole-batch cancellation are supported.
10. Per-item failure and safe retry options are shown in Activity.

Never silently drop items when queue capacity is reached.

## X post URL
1. User pastes an X status URL.
2. Intake parses supported forms and extracts a candidate post ID.
3. Resolver registry selects X adapter.
4. X adapter tries its bounded configured strategy: official API if authorized/current schema supports required data; syndication only if approved as unofficial best-effort; yt-dlp as generic extractor/fallback.
5. Resolver returns normalized media objects/variants, only including metadata actually available.
6. User chooses media and format.
7. Exactly one downloader owns acquisition: Go HTTP/aria2c for a direct resource where request requirements allow, or yt-dlp-managed download when extractor behavior is needed.
8. Result is staged and uploaded through existing TDrive pipeline.

CDN URLs can expire. On expiry, perform bounded re-resolution rather than retrying an obsolete signed URL indefinitely.

## YouTube page URL
1. User submits page URL.
2. yt-dlp metadata mode discovers formats.
3. UI displays simple default and available quality choices where practical.
4. Progressive single-file output may be handed to Go HTTP/aria2c.
5. Separate streams/extractor-dependent formats remain yt-dlp-owned.
6. FFmpeg merges/remuxes only when required and available.
7. Staged result is uploaded through existing TDrive APIs.

## Progress and Activity
Suggested stages:
- Submitted
- Validating
- Resolving
- Awaiting selection
- Queued
- Acquiring/downloading
- Processing/merging
- Ready to upload
- Uploading
- Completed / Failed / Needs attention / Cancelled

Keep stage-specific progress. Do not combine acquisition and upload percentages into a misleading overall number without a defined weighting model.

## Cancellation
Propagate cancellation to resolver HTTP/process, Go HTTP context, aria2c process tree, yt-dlp and FFmpeg. Prevent later stages from starting. Do not delete a staged file while an upload or process still owns a handle. Partial files are retained only when a safe resume contract exists.

Acquisition cancellation is not a Telegram upload failure. Upload cancellation follows TDrive's existing behavior.

## Retry and restart
Acquisition retry is distinct from upload retry. Resume partial downloads only when range and resource identity are verified. Re-resolve expired platform URLs. Discard partials if validators change.

For upload retry:
- Use existing failure classifier.
- Known not-sent/transient may be retried.
- `outcome_unknown` must not be blindly resent.
- `projection_pending` is reconciled/indexed, not re-uploaded.
- Durable multipart jobs use existing controls.

Phase 4 may add a persistent acquisition journal. Persist opaque IDs, stage/state, controlled relative path, safe validators/size and backend/variant metadata; avoid raw signed URLs, cookies or auth headers. Validate every path on restart. Never auto-repeat uncertain Telegram sends.

## Multiple media per post
Return a collection and require explicit selection when multiple objects exist. Do not silently upload every attachment.

## Destination semantics
Capture drive/channel, folder, encryption, batch grouping and user-selected media options at confirmation. Reuse the existing queue behavior for inactive drives.
