# Phase 2 — Multiple URLs and Upload Queue Integration

## Summary
Extend direct URL import to multiple URLs with review and per-item state, using TDrive's existing upload queue.

## Scope
- Multiline/multiple direct URL input.
- Stable ID per URL; do not key by filename.
- Independent validation and metadata.
- Review, remove and deselect entries.
- Capture destination drive/folder and encryption at confirmation.
- Queue selected entries in the existing queue.
- Per-item acquisition/upload progress and errors.
- Per-item and batch cancellation.
- One item failure does not abort unrelated items.
- Respect queue capacity, drive pause and source-release callbacks.

## Non-goals
Platform pages, yt-dlp extraction, authentication/cookies and media processing.

## Acceptance criteria
- Multiple direct URLs can be reviewed and queued.
- Duplicate filenames remain distinct.
- Partial success is represented per item.
- Queue-full behavior is explicit; items are not silently dropped.
- Cancellation releases only the selected item's resources.
- Existing queue serialization and drive behavior are preserved.
- E2E covers partial failure, duplicate names, queue capacity, active batch and drive switching.

Requires Phase 1.
