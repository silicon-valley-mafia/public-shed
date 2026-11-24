# Phase 4 — Acquisition Performance, aria2c, Retry and Recovery

## Summary
Harden acquisition for large/unstable downloads and introduce aria2c as an optional backend for eligible HTTP(S) resources.

## Scope
- Keep Go HTTP as a lightweight/default backend.
- Add optional aria2c acquisition.
- Define exactly one downloader owner per item, especially where yt-dlp can itself use aria2c.
- Conservative segmented-connection/retry settings.
- Validate Range support and resource identity using ETag/Last-Modified or equivalent before resume.
- Progress, cancellation and process-tree cleanup.
- Disk/concurrency limits and staging retention.
- Evaluate/implement a durable acquisition journal and restart recovery.
- Preserve TDrive upload outcome safety.

## aria2c caveats
Resume and segmentation depend on server/CDN behavior. Some sources reject ranges, expire signed URLs or throttle parallel requests. A partial filename is not proof of the same resource. Re-resolve expired URLs and discard partials if validators change. Ensure aria2c cannot bypass redirect/SSRF policy.

## Acceptance criteria
- Direct imports work without aria2c.
- Eligible downloads may use aria2c.
- Resume occurs only after range and identity checks.
- Changed resources restart safely.
- Progress and cancellation appear in Activity.
- Child processes and temporary files are controlled.
- Unknown Telegram send outcomes never cause blind reuploads.

Requires Phases 1–3.
