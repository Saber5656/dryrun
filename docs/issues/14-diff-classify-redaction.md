# 14 — Diff computation, classification, redaction

## Title
Diff computation, classification, and secret redaction

## Summary
Fill each modified/created text file's `Diff` with a unified text diff, classify binary vs
text vs too-large, and apply secret redaction so sensitive contents never appear in reports
(DESIGN §6, §10.4).

## Context
This turns raw before/after file pairs into the `Diff` values the renderers show. Redaction
is a security control (DESIGN §10.4, §10.2 T7): secret file contents must be reported as
metadata only.

## Scope
- In: `src/changes/diff.rs` (text/binary diff), `src/changes/classify.rs` (text detection,
  size caps, redaction rules). Enriches `FileChange.diff`.
- Out: engine scanning (12/21), rendering (23).

## Detailed Requirements
1. Classification for each changed regular file:
   - size > diff cap (config, default e.g. 1 MiB) → `Diff::TooLarge{size}`.
   - matches a redaction rule (path glob or content regex) → `Diff::Redacted` (never read/
     embed the body).
   - binary (NUL byte or non-text heuristic in first N KiB) → `Diff::Binary{old,new size}`.
   - else text → `Diff::Text{unified, added, removed}`.
2. Text diff via `similar`: unified format, 3 lines context, with added/removed line
   counts. For `Created`, diff against empty; for `Deleted`, the renderer shows removal
   (diff optional). Handle CRLF and no-trailing-newline correctly.
3. Redaction rules (default-on, `redact` config): default path globs from DESIGN §10.4
   (`**/.ssh/*`, `**/.aws/credentials`, `**/.netrc`, `**/*.pem`, `**/id_*`, `.env*`) plus a
   small set of content regexes (e.g. `AWS_SECRET`, `PRIVATE KEY` blocks, long high-entropy
   tokens). Configurable/extendable; on by default. A redacted file yields `Redacted` and
   only metadata (path, size, mode) elsewhere.
4. Never load a file body larger than the diff cap into memory; stream/inspect a prefix for
   classification.
5. Deterministic output (same inputs → same unified text) for snapshot tests.
6. Provide `enrich(changeset, shadow_layout, config)` that walks `FileChange`s needing a
   body (Created/Modified regular files) and sets `diff`.

## Acceptance Criteria
- A text modification yields a correct unified diff with accurate added/removed counts.
- A binary file yields `Binary` with sizes, never raw bytes.
- A file at `~/.aws/credentials`-style path or containing a private key block yields
  `Redacted`; its contents never appear in the `ChangeSet`/report (asserted by scanning the
  serialized output for the secret string — must be absent).
- Files over the cap yield `TooLarge`, not a huge diff.
- CRLF / missing-final-newline files diff without corruption.

## Validation
- `cargo test changes::diff changes::classify` including a redaction test that serializes
  the changeset and asserts the secret value is absent.

## Dependencies
08.

## Non-goals
No color/ANSI rendering (23); no scanning of trees (12/21).

## Design References
DESIGN §6 (Diff variants), §10.4 (secret handling), §10.2 T7.
