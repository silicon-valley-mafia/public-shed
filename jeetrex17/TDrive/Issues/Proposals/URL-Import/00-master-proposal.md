# Proposal: Native URL Import for TDrive

> Status: design proposal, not implemented.
> Target: [jeetrex17/TDrive](https://github.com/jeetrex17/TDrive)
> Prepared: 2026-11-24
> Source baseline inspected: master at `fc01ebf357f1c52fa513db3a6f2a59befb10c2f2`

## Executive summary

Add a native **Import from URL** workflow so users can provide direct file URLs or supported platform/page URLs, inspect what those URLs resolve to, select the intended media, and import it into the selected TDrive drive/folder without manually downloading it first.

The full target supports:
- Direct HTTP/HTTPS file links (MP4, MKV, ZIP, PDF, images and ordinary files).
- Platform URLs, initially evaluating X posts and YouTube.
- One URL or multiple URLs in a batch.
- A hybrid acquisition design: stream where compatible, otherwise stage to private temporary seekable files.
- Existing TDrive upload infrastructure for final delivery.
- Optional aria2c and FFmpeg roles, with Go HTTP and yt-dlp used according to the resource.

The design separates three concerns:

1. **Resolve:** identify the source and return normalized media metadata/variants.
2. **Acquire:** download the selected content to a controlled staging area using one explicit downloader owner.
3. **Upload:** pass the staged file into the existing TDrive upload pipeline.

This is not a proposal to replace TDrive's Telegram uploader or implement a parallel upload engine.

## Motivation and user experience

The current online-file workflow usually requires downloading a file to a normal local folder, returning to TDrive, selecting it, choosing destination/encryption, and uploading. URL import removes that manual handoff.

Target flow:

1. User chooses Import from URL.
2. Pastes one or more links.
3. TDrive validates and resolves each URL.
4. TDrive displays filename/title, type, size (if known), thumbnail, duration/dimensions and available format choices where relevant.
5. User selects desired items and confirms destination/encryption.
6. Imports enter the existing transfer queue.
7. Activity distinguishes resolving, acquiring, processing, ready-to-upload, uploading and terminal states.
8. Temporary files are cleaned only after no longer needed for upload or safe recovery.

Simple direct-file imports should not expose extractor/downloader internals to the user.

### Direct URL examples

- X CDN video: `https://video.twimg.com/amplify_video/2124577617148030976/vid/avc1/720x898/TAYkfCibgRm7FmDJ.mp4`
- Internet Archive MP4: `https://ia801006.us.archive.org/27/items/electricsheep-flock-248-32500-3/00248%3D32663%3D28059%3D27827.mp4`

### Platform URL examples

- X post/status URL.
- YouTube video URL.
- Other supported media pages as adapters are added and validated.

## Product decisions from the discussion

| Topic | Current decision |
|---|---|
| URL coverage | Both direct links and platform URLs |
| Batch | Multiple URLs |
| Transfer model | Hybrid streaming + private temporary staging |
| Upload | Existing TDrive upload engine |
| Platform extractor | Resolver registry, yt-dlp candidate |
| X-specific behavior | Investigate structured post/media resolution, not just raw page download |
| Basic direct download | Go HTTP |
| High-performance download | aria2c optional |
| Media processing | FFmpeg optional |
| Authentication | Cookies/login deferred |
| Delivery | One master proposal and five implementation phases |

## Architecture overview

The resolver returns one or more normalized media items. The acquisition manager chooses exactly one owner for each download. The resulting file is staged privately and sent through the existing upload path.

See [01-architecture.md](01-architecture.md) for components and diagrams, and [02-workflow.md](02-workflow.md) for end-to-end scenarios.

### X: lesson from Grok

Grok explained that its internal X post-fetching capability obtains a structured post object, including media metadata, and exposes direct CDN variants under `video.twimg.com`. The useful architectural lesson is that **resolving a post into media objects is distinct from downloading the media**.

TDrive cannot call Grok's internal capability. It can implement the same general pattern using permitted/testable adapters:

1. Official X API v2, when current endpoint schema, account entitlement and authorization permit the required fields.
2. Public syndication endpoint (`https://cdn.syndication.twimg.com/tweet-result?id={TWEET_ID}&lang=en`) as an explicitly unofficial, unstable, best-effort option.
3. yt-dlp X extractor as a general-purpose fallback candidate.

Important caveat: the sample API request in the discussion used `media.fields=variants`. Do not assume this field is currently available on every endpoint/plan. Verify official current documentation and an actual authorized response. Do not make syndication a sole dependency. Avoid private GraphQL reverse engineering and brittle HTML scraping as a production foundation.

### Tool responsibilities

- **Go HTTP:** lightweight default for direct file downloads; bounded streaming to disk.
- **yt-dlp:** platform metadata/format extraction and, where necessary, managed acquisition. It may return direct URLs or own a platform-specific transfer.
- **aria2c:** optional high-performance acquisition for eligible HTTP(S) resources, especially where segmented download/resume is supported.
- **FFmpeg:** optional merge/remux/audio processing only when required or selected.
- **Cobalt:** product/architecture reference for paste-link UX and resolver response concepts, not an assumed hosted dependency.

One item must have exactly one acquisition owner. yt-dlp can itself use aria2c as an external downloader; do not also start a second independent Go/aria2 download for the same item.

## Why staging is necessary

TDrive's current uploader uses seekable source boundaries for size validation, encryption, multipart planning and resumable cases. A raw network response body cannot be passed through every existing path as-is. Private temporary staging provides a seekable source without requiring a user-visible permanent download.

The staged path is then passed into the existing TDrive app/upload APIs. Reuse destination, encryption, upload limits, multipart/resumable handling, progress, Activity, Telegram publication and local projection. Preserve existing failure classification, especially the rule that an unknown Telegram send outcome must not be blindly retried.

## Five-phase roadmap

| Phase | Name | Main result |
|---|---|---|
| 1 | Direct HTTP/HTTPS URL Import | One direct file URL end to end |
| 2 | Multiple URLs and Upload Queue Integration | Review and queue batches of direct URLs |
| 3 | Platform URL Resolution and Media Extraction | X/YouTube resolver adapters and normalized metadata |
| 4 | Acquisition Performance, aria2c, Retry and Recovery | Optional aria2c, safe resume and staging recovery |
| 5 | Advanced Media Selection and Optional Processing | Multi-item/quality/audio selection and optional FFmpeg |

Phase 1 starts with Go HTTP and a single URL. Phase 2 adds direct URL batches. Phase 3 introduces platform extraction. Phase 4 adds resilient acquisition and aria2c. Phase 5 exposes advanced media choices and optional processing.

Detailed phase issue drafts are under [phase-issues/](phase-issues/README.md). Full sequencing is in [05-roadmap.md](05-roadmap.md).

## Security and scope boundaries

Because this feature fetches arbitrary user-supplied URLs and may invoke child processes, it must address SSRF, DNS rebinding, redirect validation, private/link-local/metadata destination blocks, process security, filename sanitation, file-size/time/disk limits, cancellation, temp-file lifecycle and secret redaction.

Initial non-goals:
- Authenticated downloads, cookie import and login sessions.
- DRM circumvention or access-control bypass.
- Mandatory transcoding.
- Permanent server-side media cache.
- Hosted Cobalt API dependency without permission.
- Replacing the TDrive upload engine.
- Guaranteeing every platform URL is downloadable.

See [04-security-and-reliability.md](04-security-and-reliability.md).

## Upstream codebase baseline

The current upstream repository was checked on 2026-11-24. Observed `master` head: `fc01ebf357f1c52fa513db3a6f2a59befb10c2f2`. This commit is the merge commit for PR #200. Earlier conversation notes called PR #200 a draft; that status is stale.

Relevant paths:
- `backend/services/file/upload.go`
- `backend/services/file/upload_failures.go`
- `internal/app/app.go`
- `frontend/src/modules/transfers.ts`
- `frontend/src/modules/upload-queue.ts`
- `frontend/src/api/files.ts`
- `docs/architecture/transfers-and-mounts.md`
- `backend/updater/download.go`
- `frontend/e2e/upload-queue.spec.ts`

See [03-research-and-codebase-baseline.md](03-research-and-codebase-baseline.md). This was a feature-focused inspection, not an exhaustive read of every file. Revalidate the baseline before implementation.

## Document package

- [Master proposal](00-master-proposal.md)
- [Architecture](01-architecture.md)
- [Workflow](02-workflow.md)
- [Codebase baseline and research](03-research-and-codebase-baseline.md)
- [Security and reliability](04-security-and-reliability.md)
- [Roadmap](05-roadmap.md)
- [Decisions and open questions](06-decisions-and-open-questions.md)
- [Verification matrix](07-verification-matrix.md)
- [Five phase issue drafts](phase-issues/README.md)

## Publication and implementation rule

Publish the master tracking issue first, then each phase issue in order. Implement and review one phase at a time; do not provide all five phases as a single coding task. Link each phase issue to the master and its predecessor. Keep docs synchronized with maintainer decisions.

No upstream TDrive issues have been created by this proposal package. No TDrive source code has been modified.
