# Verification matrix

## Phase 1 — Direct URL
| Scenario | Expected |
|---|---|
| Valid HTTPS direct file | Accepted |
| Unsupported scheme | Rejected |
| Redirect to public host | Allowed only after validation |
| Redirect to loopback/private/link-local/metadata | Blocked |
| DNS rebinding/private answer | Blocked |
| Missing Content-Length | Actual byte cap still enforced |
| False Content-Length | Actual bytes determine limit |
| Non-success response/error HTML | Structured failure; not uploaded as intended file |
| Malicious Content-Disposition | Filename sanitized |
| Disk full/permission failure | Clear failure and cleanup |
| Cancel during download | Request stops, partial handled safely |
| Upload integration | Existing API receives correct staged path/destination/encryption |
| Unknown Telegram outcome | No blind resend |

## Phase 2 — Batch
| Scenario | Expected |
|---|---|
| Several URLs | Stable unique item IDs |
| One invalid among valid | Valid entries remain selectable |
| Duplicate filenames | Items remain independent |
| Queue active/full | Existing serialization; explicit backpressure |
| Drive switch | Existing pause/admission policy |
| One acquisition fails | Others continue |
| Cancel one/batch | Only owned work stops |
| Partial success | Per-item status correct |

## Phase 3 — Platforms
| Scenario | Expected |
|---|---|
| Public X video | Normalized media result |
| X post with multiple media | Selectable individual items |
| X GIF-like media | Actual source format handled |
| Private/deleted/unavailable | Clear error |
| Official X API field/entitlement missing | Limitation or approved fallback |
| Syndication rate-limit/change | Bounded fallback/error |
| YouTube progressive format | Format metadata available |
| Separate audio/video | Merge requirement represented |
| yt-dlp missing/incompatible | Actionable dependency error |
| Signed URL expires | Bounded re-resolution |

## Phase 4 — Acquisition
| Scenario | Expected |
|---|---|
| aria2c installed | Eligible item can use it |
| aria2c missing | Go HTTP remains functional |
| Range supported | Resume only after identity validation |
| Range unsupported | Restart safely |
| Validator changes | Partial discarded |
| Child process cancelled | Process tree exits |
| Concurrent progress | Correct item mapping |
| Disk/concurrency limit | Clear backpressure |
| Restart recovery | Paths/source metadata validated |
| Unknown Telegram send | No fresh upload |

## Phase 5 — Media/processing
| Scenario | Expected |
|---|---|
| Multiple variants | Only actual choices shown |
| Multiple post items | User chooses items |
| Audio-only | Works where available |
| Separate audio/video | FFmpeg merges when required |
| FFmpeg missing | Processing limitation; direct MP4 still works |
| Direct MP4 | No FFmpeg requirement |
| Remux possible | Prefer stream copy |
| Unsafe filename | Safe TDrive name |

## Test layers
- Unit: URL parsing, host policy, SSRF addresses, redirects, normalized metadata, variant identity, filename sanitation, backend selection, resume validator and cleanup.
- Integration: controlled local HTTP server for redirects, Range/ETag, inaccurate Content-Length, slow body, cancellation and disk errors; stub resolver and child processes.
- Upload integration: fake Telegram client verifies bytes, destination, encryption and correct existing API path.
- E2E: direct single, direct batch, active queue, partial failure, per-item/batch cancel, drive switch, X/YouTube resolution and selection, restart if journal exists.
- Regression: local uploads, folder imports, encryption, multipart/resumable, queue serialization, cancellation and outcome-unknown handling.
