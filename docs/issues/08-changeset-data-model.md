# 08 — ChangeSet data model

## Title
ChangeSet data model

## Summary
Define the `ChangeSet` and all member types from DESIGN §6 with deterministic ordering and
serde derives, the shared vocabulary every engine produces and the report layer consumes.

## Context
The ChangeSet is the product's core artifact (DESIGN §6). It must be engine-agnostic,
deterministic (DESIGN §6.2), and serde-serializable so JSON (22) and human (23) renderers
share one source.

## Scope
- In: `src/changes/mod.rs` — `ChangeSet`, `FileChange`, `ChangeKind`, `Meta`, `Diff`,
  `BlockedWrite`, `NetworkAttempt`, plus a `summary()` and a canonical `sort()`.
- Out: diff computation (14), engine-specific scanning (12/21), serialization schema (22).

## Detailed Requirements
1. Types exactly as DESIGN §6 (field names and variants must match, since 22's JSON schema
   is derived from them):
   - `ChangeSet { engine, root, files, blocked_writes, network_attempts, truncated,
     caveats }`.
   - `FileChange { path, kind, old_meta, new_meta, diff }` (`path` relative to `root`).
   - `ChangeKind { Created, Modified, Deleted, Renamed{from}, ModeChanged, TypeChanged,
     SymlinkChanged }`.
   - `Meta { size, mode, is_symlink, symlink_target, content_hash }` (hash =
     `Option<[u8;32]>` blake3).
   - `Diff { Text{unified,added,removed}, Binary{old_size,new_size}, TooLarge{size},
     Redacted }`.
   - `BlockedWrite { path, op }`, `NetworkAttempt { summary }`.
2. serde derives on all; `content_hash` serialized as lowercase hex string (custom
   serializer) or omitted when `None`. Enum tagging for `ChangeKind`/`Diff` chosen to
   match DESIGN §7 JSON (document the mapping; e.g. `kind:"renamed"` with a `from` field).
3. `ChangeSet::summary() -> Summary` counts per DESIGN §7 `summary` block
   (created/modified/deleted/renamed/mode_changed/blocked_writes/network_attempts).
4. `ChangeSet::sort()`: sort `files` by `path` (byte order), `blocked_writes` by path,
   `network_attempts` by summary — the determinism requirement (DESIGN §6.2). Engines call
   this before returning.
5. Builder/helpers: `push_file`, `push_blocked`, `push_network`, `mark_truncated`,
   `add_caveat`, so engines construct changesets uniformly.
6. No I/O in this module (pure data + ordering).

## Acceptance Criteria
- All types compile and serialize; a snapshot test serializes a hand-built `ChangeSet` and
  matches a committed fixture whose shape equals DESIGN §7's `changes` block.
- `sort()` produces identical output regardless of insertion order (test).
- `summary()` counts match a fixture with known contents.
- `content_hash` serializes as hex and is omitted when absent.

## Validation
- `cargo test changes::mod` (serde snapshot via `insta` + sort/summary unit tests).

## Dependencies
01.

## Non-goals
No diff generation, no filesystem scanning, no JSON schema file (22).

## Design References
DESIGN §6 (model), §6.2 (determinism), §7 (JSON mapping).
