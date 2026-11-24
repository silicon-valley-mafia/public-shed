# Decisions and open questions

## Recorded decisions

| Topic | Position |
|---|---|
| Import entry points | Direct/platform URLs, browser drag-and-drop and clipboard paste |
| Browser File/Blob | Stage through bounded streaming/chunked bridge |
| URL-only browser payload | Classify as URL and route through backend SSRF controls |
| Clipboard access | User-initiated paste event only; no background polling |
| Common pipeline | All inputs use normalized review items and existing queue/uploader |
| Rollout | Six phases, including browser intake |

| Topic | Position |
|---|---|
| URL scope | Direct file URLs and platform/page URLs |
| Batch | Multiple URLs |
| Acquisition | Hybrid stream + private staging |
| Final upload | Existing TDrive uploader |
| Platform extraction | Resolver registry; yt-dlp candidate |
| X | Consider official API, syndication best-effort and yt-dlp |
| Basic download | Go HTTP |
| Performance backend | aria2c optional |
| Media processing | FFmpeg optional |
| Authentication/cookies | Deferred |
| Rollout | Five phases plus master proposal |

## Responsibility boundaries
- Go HTTP: baseline direct download.
- yt-dlp: extraction/metadata and sometimes acquisition ownership.
- aria2c: optional high-performance HTTP(S) downloader; may be invoked under yt-dlp, but never as a second independent download.
- FFmpeg: merge/remux/selected processing only.
- Cobalt: UX/architecture reference unless permitted self-hosting is approved.

## Browser intake questions
1. Which Wails frontend/runtime versions expose the required DataTransfer and ClipboardEvent behavior consistently?
2. Which bridge mechanism transfers large File/Blob payloads without base64 expansion or renderer memory spikes?
3. Can backend staging accept bounded chunked writes with cancellation and per-session limits?
4. Which drag payloads expose bytes versus text/uri-list or plain text only?
5. How should paste shortcuts be scoped so ordinary text fields remain unaffected?

## Open questions for maintainer/prototype review
1. Where should Import from URL appear in UI?
2. Wails synchronous resolver call or asynchronous job/events?
3. Should syndication be shipped or remain experimental?
4. What X API credential/entitlement model is acceptable?
5. How should yt-dlp be bundled, installed, updated and versioned on each OS?
6. What JS runtime/dependencies are required for supported yt-dlp/YouTube behavior?
7. aria2c integration via CLI/RPC or alternative?
8. For which resources should yt-dlp own download versus return a URL to Go HTTP/aria2c?
9. Per-host/global concurrency and disk limits?
10. Temp retention duration and cleanup policy?
11. Is persistent acquisition recovery mandatory in Phase 4?
12. Keep Phase 3 selection simple and defer full format picker to Phase 5?
13. What file-signature/probe policy applies to generic documents?
14. Which redacted diagnostics are useful without storing signed URLs/post content?
15. Which optional binaries are supported and how do users install/update them?

## Recommendations
- Browser inputs should be implemented after the normalized intake and queue contracts from Phases 1–2 stabilize.
- Never read clipboard in the background; use a user-initiated paste event.
- URL-only drags must pass through backend SSRF validation.
- Use bounded File/Blob streaming/chunks rather than whole-file base64.

- Phase 1 Go HTTP only.
- Phase 2 direct URLs only.
- Phase 3 platform resolver registry.
- Do not make syndication a hard dependency.
- Let yt-dlp own transfers only when extractor behavior requires it.
- Add aria2c conservatively in Phase 4.
- FFmpeg optional.
- Auth/cookies separate future scope.
- Existing Telegram outcome classifier remains authoritative.
