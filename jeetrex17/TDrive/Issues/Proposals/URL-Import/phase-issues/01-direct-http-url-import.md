# Phase 1 — Direct HTTP/HTTPS URL Import

## Summary
Add an **Import from URL** action so a user can paste one direct HTTP/HTTPS file URL and import it without manually downloading it to a user-visible folder.

## User flow
1. Paste one direct URL.
2. TDrive validates it and resolves filename/type/size where available.
3. User confirms destination and encryption settings.
4. Go HTTP downloads to private temporary seekable storage.
5. The staged file enters the existing TDrive upload pipeline.
6. Activity distinguishes acquisition and upload outcomes.

## Scope
- One HTTP/HTTPS direct file URL.
- URL/redirect/host safety and SSRF validation.
- Basic metadata retrieval.
- Bounded streaming Go HTTP client.
- Private temporary staging, filename sanitation and actual byte limits.
- Cancellation and cleanup/retention.
- Existing upload API integration and destination/encryption preservation.
- Existing multipart/resumable/progress/failure behavior.

## Non-goals
Platform page extraction, batch URLs, yt-dlp/aria2c requirements, FFmpeg, authenticated sources/cookies.

## Constraints
Do not create a second Telegram uploader. Existing upload paths use seekable sources in important cases, so stage when required. Preserve Telegram outcome-unknown and projection-pending safeguards.

## Acceptance criteria
- One direct URL imports without manual download.
- Unsafe schemes/redirects/targets are rejected.
- Missing or false Content-Length cannot bypass the maximum byte limit.
- Large resources are streamed to disk, not held in memory.
- Existing encryption and uploader path are used.
- Cancellation and failure clean/retain partials by an explicit rule.
- Tests cover valid response, redirects/SSRF, errors, oversized body, unknown size, cancellation, disk failure and upload integration.

Requires no earlier phase.
