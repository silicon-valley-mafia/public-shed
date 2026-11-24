# Phase 02 — Individual file retry

**User-visible capability:** Retry one eligible failed file directly from its row in Activity.

This is the smallest retry action and establishes the per-file state model reused by folder and batch actions.

## Issues

1. [02A — Track retry state per failed source](issue-02A-per-file-retry-state.md)
2. [02B — Add Try Again to an individual failed file](issue-02B-individual-retry-action.md)

## Completion criteria

- One action selects one stable source identity.
- Duplicate filenames cannot select the wrong file.
- Other failed, successful and queued files are unaffected.
- The original report remains accurate after a partial retry.
