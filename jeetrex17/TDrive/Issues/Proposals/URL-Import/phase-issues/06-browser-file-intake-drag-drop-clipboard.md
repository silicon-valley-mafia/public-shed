## Feature: Browser file intake through drag-and-drop and clipboard paste

### Summary
Extend TDrive's unified import experience so users can drag browser-visible images/files into the app and paste copied images/files from the clipboard, without first saving them to a conventional folder. All accepted content must use the common import review model, existing queue and existing upload pipeline.

### User stories
- As a user, I can drag one or more files from the browser into an explicit TDrive import drop zone.
- As a user, I can drag an image from a webpage; if the browser exposes file bytes, TDrive stages those bytes, and if it exposes only a URL, TDrive identifies it as a URL candidate.
- As a user, I can copy an image and press Ctrl+V / Cmd+V while the import surface is focused to add it.
- As a user, I can paste a file when the browser exposes a clipboard file item.
- As a user, ordinary paste behavior outside the import surface remains unchanged.

### Functional requirements
1. Add a dedicated drop target with clear drag-active and unsupported-payload feedback.
2. Inspect `DataTransfer.files` for actual file bytes and support multiple files.
3. Inspect `DataTransfer.items`, `text/uri-list` and `text/plain` for URL/link payloads when bytes are unavailable.
4. Handle clipboard `paste` events only in the import surface; accept image/file items exposed by `clipboardData.items/files`.
5. Recognize URL text pasted into the import surface as a URL candidate; do not treat arbitrary text as a file.
6. Normalize all items to stable IDs, source kind, safe suggested name, MIME/size hints, preview and validation state.
7. Stage File/Blob data into app-managed private temporary storage through a bounded streaming/chunked frontend/backend bridge. Do not send large files as a single base64 data URL.
8. Enforce item count, per-file and aggregate byte limits, disk limits, cancellation and staging ownership in the backend.
9. Treat browser filename/MIME as untrusted hints; sanitize names and validate actual content according to the file policy.
10. Route URL-only drag/clipboard inputs through the existing backend URL resolver and SSRF protections. Never fetch these directly from renderer code.
11. Reuse the existing destination, encryption, queue, Activity, uploader and source-release contracts.

### Non-goals
- Background clipboard monitoring or reading clipboard without a user gesture.
- Arbitrary webpage scraping or automatically importing every image on a page.
- Browser extension integration.
- Persisting renderer object URLs as durable file references.
- A new uploader implementation.

### Acceptance criteria
- Single and multiple file drops are staged and queued correctly.
- Browser image drag works for both File payloads and URL-only payloads with distinct behavior.
- Clipboard image paste and exposed clipboard file paste work.
- Plain text paste outside the import surface is unaffected.
- Empty, denied and unsupported clipboard cases fail gracefully.
- Large files use bounded/chunked transfer and do not cause unbounded renderer memory growth.
- MIME spoofing, malicious filenames, oversized files and excessive item counts are rejected or handled safely.
- Cancellation, cleanup, duplicate names, object URL revocation and source release are covered.
- Browser-originated items use the existing TDrive uploader and preserve destination/encryption semantics.
- E2E tests cover drag/drop, clipboard, URL-only drops and regressions to normal uploads.

### Implementation notes
Browser behavior varies. A drag of an image element may expose a File, a URL, or both depending on browser/source. Clipboard APIs are permission- and gesture-sensitive; use the paste event's DataTransfer payload rather than background clipboard polling. Confirm actual behavior in supported Wails runtime/browser versions before finalizing the UI contract.
