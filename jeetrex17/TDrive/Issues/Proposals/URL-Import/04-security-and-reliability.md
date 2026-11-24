# Security, reliability and operations

## Threat model
URL import makes TDrive fetch user-supplied locations and may invoke external tools. Main risks: SSRF; DNS rebinding and redirect SSRF; external downloader policy bypass; disk/memory/bandwidth/CPU exhaustion; malicious filenames; malformed media; process injection; leakage of signed URLs/cookies/headers; invalid resume; duplicate Telegram uploads after uncertain acceptance.

## URL and network controls
- HTTP/HTTPS only initially.
- Reject URL user-info unless a future authenticated-source design explicitly supports it.
- Validate parsed host, port and resolved addresses; no hostname prefix checks as access control.
- Block IPv4/IPv6 loopback, private, link-local, multicast, unspecified/reserved, mapped IPv6 and known metadata ranges.
- Resolve and validate DNS, then pin/enforce approved address for request while retaining TLS hostname verification.
- Revalidate on new connections and every redirect.
- Bound redirects; strip sensitive headers on cross-origin redirects.
- Ensure aria2c/yt-dlp cannot independently resolve unrestricted hosts. If equivalent policy cannot be enforced, apply OS/network egress restrictions or do not permit that path.
- Avoid logging query-bearing signed URLs.

## Resource limits
Configure URL count, URL length, resolver deadline, connect/header/body idle timeout, total download deadline, redirect count, actual bytes, per-file/batch disk quota, global/per-host concurrency, aria2c splits, process count, retry count and FFmpeg output/time limits. Content-Length is advisory; enforce actual bytes written. Handle disk-full and permission errors.

## Filename and content safety
- Treat Content-Disposition and extractor titles as untrusted.
- Strip path separators, traversal, control characters and reserved names.
- Use app-generated storage names; remote filename is display metadata only.
- Do not trust MIME/content type; inspect response and signatures/probe where appropriate.
- Never execute downloaded content.
- Keep processor I/O inside transfer-owned staging directory.

## Staging
- App-managed private temp root with restrictive permissions.
- Random transfer directories and generated filenames.
- Unique .part and completed paths; atomic completion.
- Validate final path remains under owned directory.
- Do not delete while downloader/processor/uploader has a handle.
- Track every partial/processed artifact.
- Define cleanup and retention on success, cancel, failure, expiry and shutdown.

## Resume correctness
Resume only when Range is supported and resource identity is verified. Prefer ETag/Last-Modified and validate 206/Content-Range semantics. Re-resolve expired signed URLs. If validators change, discard partial and restart. A filename or URL string alone is not identity. Local hash records bytes but does not authenticate origin without trusted digest.

## External process management
For yt-dlp, aria2c and FFmpeg:
- Use argument arrays, never shell interpolation.
- Resolve executable path deterministically and check versions.
- Use controlled working directory/environment/output path.
- Drain stdout/stderr with bounded buffers.
- Cancel and reap processes; handle process trees on Windows and Unix.
- Bound concurrent children and never leak credentials in process arguments/logs.

## Upload outcome safety
Use TDrive's failure classifier:
- Known-not-sent/transient may be retryable.
- `outcome_unknown` must not be blindly resent.
- `projection_pending` means remote accepted/local projection pending; reconcile, do not upload again.
- Resumable multipart keeps existing controls.
Acquisition retry is a distinct stage and must not trigger unsafe Telegram resend.

## Credentials and cookies
Authenticated platform downloads and cookie import are deferred. Future support needs explicit consent, secure credential storage, redaction, cross-origin header stripping, platform-specific compliance review, expiry/revocation handling.

## Privacy
Persist safe display metadata only. Avoid raw signed CDN URLs, cookies, authorization headers or unnecessary post content in Activity/history/logging. Redact sensitive query parameters.

## Restart and recovery
Phase 4 may add a journal with opaque transfer ID, state, controlled relative path, safe validators/size, backend and selected variant. Validate all paths under the app temp root on restart. Do not assume child processes survive. Never auto-repeat uncertain Telegram sends. Without a journal, clearly mark staging as session-only and clean orphans under documented rules.
