# Phase 3 — Platform URL Resolution and Media Extraction

## Summary
Support media-page URLs, initially X posts and YouTube, via a reusable resolver registry and normalized media metadata.

## Scope
- Resolver interface/registry and normalized media/variant schema.
- X adapter evaluating official API v2 when authorized/current schema supports the required fields; public syndication only as an explicitly unstable best-effort option; yt-dlp extractor as a candidate fallback.
- yt-dlp metadata/format extraction for supported platforms.
- Title/name, type, thumbnail, duration, dimensions, formats and size only when actually available.
- Multiple media objects and basic selection.
- Resolve without uploading; route selected items into the common acquisition and existing upload path.
- Decide how yt-dlp and runtime dependencies are installed/versioned.

## X caveats
Grok described an internal X post-fetch path that exposes structured post metadata and direct `video.twimg.com` variants. TDrive cannot call that internal capability; it should implement the general resolve-then-acquire pattern. Do not assume `variants` is available from every X API endpoint or plan. Verify official schema and actual authorized responses. Syndication is undocumented/unofficial; never make it the sole dependency. Avoid private GraphQL reverse engineering as the production foundation.

## Non-goals
Authenticated downloads/cookie import, DRM/paywall/access-control bypass, mandatory FFmpeg and full advanced format UX.

## Acceptance criteria
- Representative public X and YouTube examples resolve.
- Resolver returns normalized results without upload side effects.
- User can select a media item and send it through the existing transfer flow.
- Multiple items/formats have distinct IDs.
- Unsupported/private/deleted/unavailable sources return actionable errors.
- Fallback is bounded, observable and tested.

Requires Phases 1 and 2.
