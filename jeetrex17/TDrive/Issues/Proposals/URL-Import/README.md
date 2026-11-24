# TDrive — URL Import Proposal

This folder contains a unified import proposal for URLs, browser drag-and-drop and clipboard-pasted files/images.

## Start here

- [Master proposal](00-master-proposal.md)
- [Architecture](01-architecture.md)
- [End-to-end workflow](02-workflow.md)
- [Research and codebase findings](03-research-and-codebase-baseline.md)
- [Security, reliability and operational requirements](04-security-and-reliability.md)
- [Six-phase roadmap](05-roadmap.md)
- [Phase issue drafts](phase-issues/README.md)
- [Decision log and open questions](06-decisions-and-open-questions.md)
- [Verification and acceptance matrix](07-verification-matrix.md)

## Related TDrive proposal: failed-upload recovery

The earlier failed-upload recovery workspace has been consolidated here as a sibling proposal, preserving its six phases, issue drafts, research, architecture, workflow and verification documents:

- [Failed-upload recovery proposal](retry-recovery/README.md)
- [Retry architecture](retry-recovery/architecture.md)
- [Retry roadmap](retry-recovery/plan.md)
- [Retry research](retry-recovery/research.md)
- [Retry phase issue drafts](retry-recovery/phases/)
- [Retry verification matrix](retry-recovery/verification/acceptance-matrix.md)

## Scope at a glance

The proposal covers:
- Direct HTTP/HTTPS file URLs.
- Browser drag-and-drop for files/images and URL-only links.
- Clipboard paste for browser-exposed image/file items.
- Platform URLs, initially evaluating X and YouTube.
- X-specific post/media resolution strategies and yt-dlp.
- Batch URL imports.
- Hybrid acquisition using Go HTTP, optional aria2c, and platform-specific downloader ownership where necessary.
- Private temporary staging and integration with TDrive's existing upload pipeline.
- Optional FFmpeg processing.
- Six sequential implementation phases, including browser-native intake.

Authentication/cookies are deferred. No implementation is claimed by these documents.

## Baseline

The TDrive source baseline was checked at commit [fc01ebf](https://github.com/jeetrex17/TDrive/commit/fc01ebf357f1c52fa513db3a6f2a59befb10c2f2), which is the current `master` commit observed during preparation (2026-11-24). Relevant implementation details and caveats are captured in the research document. Revalidate the baseline immediately before implementation.

## Issue publication status

The phase issue files are drafts intended to be reviewed and then posted individually to the upstream TDrive repository. Creating this proposal package does not create upstream issues or change TDrive source code.
