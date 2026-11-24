# Phase 5 — Advanced Media Selection and Optional Processing

## Summary
Add advanced selection for platform media, including multiple items, format/quality, audio-only and optional FFmpeg processing.

## Scope
- Choose individual media items from a post/page.
- Display real available quality/format variants.
- Audio-only where supported.
- Handle separate audio/video streams.
- Optional FFmpeg/ffprobe for merging/remuxing.
- Prefer stream copy; avoid unnecessary transcoding.
- Preserve metadata and safe filenames.
- Extend provider adapters incrementally.

## Non-goals
Authenticated cookies, DRM circumvention, paywall bypass and mandatory transcoding.

## Acceptance criteria
- Users can select among available media items and formats.
- Audio-only works where the resolver provides a valid option.
- Separate audio/video merge works when FFmpeg is available.
- Direct MP4 does not need FFmpeg.
- Missing tools/errors are clearly reported.
- Processing state is distinct from acquisition and upload.
- No lossy re-encoding unless explicitly requested or required.

Requires Phase 3; reuse Phase 4 acquisition/recovery components where appropriate.
