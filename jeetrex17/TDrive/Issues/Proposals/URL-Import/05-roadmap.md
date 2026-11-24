# Five-phase roadmap

## Overview

| Phase | Name | Outcome | Dependency |
|---|---|---|---|
| 1 | Direct HTTP/HTTPS URL Import | One direct file URL end-to-end | None |
| 2 | Multiple URLs and Upload Queue Integration | Batch direct URL review/queue | Phase 1 |
| 3 | Platform URL Resolution and Media Extraction | X/YouTube resolvers and normalized metadata | Phases 1–2 |
| 4 | Acquisition Performance, aria2c, Retry and Recovery | Optional aria2c, safe resume/recovery | Phases 1–3 |
| 5 | Advanced Media Selection and Optional Processing | Variants, multi-item, audio-only and optional FFmpeg | Phase 3; reuse Phase 4 acquisition |

## Phase 1 — Direct HTTP/HTTPS URL Import
Goal: one direct file URL imported end-to-end.

Scope: Import from URL action; one HTTP/HTTPS file URL; validate URL/redirect/destination; retrieve filename/type/size where available; Go HTTP bounded streaming; private temporary seekable staging; actual byte/time/disk limits; use existing upload APIs; reuse drive/folder/encryption/multipart/resumable/progress/failure; cancellation and cleanup.

Non-goals: platform pages, batch URLs, aria2c, FFmpeg, cookies.

Acceptance: direct URL imports without manual download; unsafe redirects/targets rejected; absent/false Content-Length cannot bypass cap; bounded memory; existing uploader used; cleanup tested.

## Phase 2 — Multiple URLs and Upload Queue Integration
Goal: multiple direct URLs with review and item-level status.

Scope: multiline input, stable IDs, independent resolution, review/remove/deselect, capture drive/destination/encryption, existing queue integration, acquisition/upload progress, independent failures, per-item/batch cancel, queue capacity and drive pause behavior.

Non-goals: platform pages, authenticated sources, processing.

Acceptance: multiple URLs review and queue; duplicate filenames remain distinct; partial success visible; queue capacity explicit; cancellation releases owned resources; E2E covers partial failure, duplicate names, active queue and drive switch.

## Phase 3 — Platform URL Resolution and Media Extraction
Goal: platform URLs, initially X and YouTube.

Scope: resolver registry and normalized media/variant model; X adapter evaluates official API when authorized/schema supports fields, syndication only as best-effort unofficial and yt-dlp fallback; yt-dlp metadata/format extraction; metadata where available; multiple media objects and basic selection; dependency packaging/version decision; selected media routed into common acquisition and uploader.

Caveats: verify X `variants` field against live docs/authorized response; syndication can break; do not rely on Grok internal capability or private GraphQL; auth/cookies deferred.

Acceptance: representative public X/YouTube resolve; normalized result has no upload side effects; selected item uses existing pipeline; unavailable/private/deleted errors clear; bounded fallback.

## Phase 4 — Acquisition Performance, aria2c, Retry and Recovery
Goal: reliable large downloads.

Scope: Go HTTP remains baseline; optional aria2c for suitable sources; deterministic downloader ownership (yt-dlp may itself invoke aria2c); conservative segmented/retry settings; Range and ETag/Last-Modified checks; progress/cancellation/process cleanup; disk/concurrency limits; temp retention; evaluate/implement acquisition journal; preserve Telegram uncertainty behavior.

Acceptance: no aria2c needed for basic use; resume only with verified source and range support; changed source restarts safely; progress/cancel visible; no orphan processes/temp leaks; unknown Telegram send never blindly reuploaded.

## Phase 5 — Advanced Media Selection and Optional Processing
Goal: richer media choices.

Scope: select individual post media items, quality/container variants, audio-only where available, separate audio/video, optional FFmpeg merge/remux, prefer stream copy, preserve metadata/safe names, extend platforms incrementally.

Acceptance: real variants only; multiple item/audio choices; FFmpeg used only when required; direct MP4 works without FFmpeg; processing state distinct; no unnecessary lossy transcoding.

## Delivery rule
Publish master issue first. Implement/review one phase at a time. Do not send all five phases to an implementation agent as one task.
