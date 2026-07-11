# 12 — Tree-diff changeset scanner (copy engines)

## Title
Tree-diff changeset scanner (copy engines)

## Summary
Implement the scanner that diffs the pre-run manifest (11) against the post-run shadow tree
to produce a `ChangeSet` (created/modified/deleted/renamed/mode/type/symlink) for the
macOS and portable engines (DESIGN §6.1 copy engines).

## Context
Copy engines have no kernel changelog; they reconstruct the ChangeSet by comparing before
(manifest) and after (shadow) states (DESIGN §6.1). Rename detection is a documented
best-effort heuristic (DESIGN §3.5).

## Scope
- In: `src/engine/common/scan.rs` — `scan(shadow: &ShadowLayout, manifest: &Manifest,
  policy) -> ChangeSet` (files + kinds + hashes; diff bodies filled by 14).
- Out: text/binary diff generation (14), overlay whiteout scan (21), blocked-write/network
  channels (engine-specific).

## Detailed Requirements
1. Walk the post-run shadow tree deterministically (sorted). For each path compute `Meta`
   (size, mode, symlink, blake3 hash within cap).
2. Classify against the manifest:
   - present after, absent before → `Created`.
   - present both, hash differs (or symlink target differs) → `Modified` /
     `SymlinkChanged`.
   - present both, content identical, mode/owner differs → `ModeChanged`.
   - regular↔symlink or file↔dir change → `TypeChanged`.
   - present before, absent after → candidate `Deleted`.
3. Rename heuristic: for each `Deleted` candidate D and each `Created` C, if
   `hash(D) == hash(C)` (both hashed, non-empty) and sizes match, pair them as
   `Renamed{from: D.path}` on C and drop the separate Deleted/Created. Ambiguous many-to-
   many matches stay as separate Deleted+Created and a caveat is added. Always append the
   caveat `"renames are best-effort"`.
4. Respect `max_files`: stop after N changes, set `truncated=true`, add a caveat.
5. Translate shadow paths back to workspace-relative paths via 11's map; the `root` is the
   workspace.
6. Populate `FileChange.old_meta`/`new_meta`; leave `diff: None` (14 fills text/binary/
   redaction later in the pipeline, or 14 is called here — decide: this issue produces
   metadata + kinds, 14 enriches `diff`). Document the hand-off precisely so 14 knows which
   changes need bodies (Modified/Created text files).
7. Handle empty files (hash of empty = still hashable), zero-length rename ambiguity
   (do not pair empties), and unreadable files (record as changed with a caveat, never
   panic).
8. Add the caveat `"command ran at a shadow path"` (DESIGN §3.5) for copy engines.

## Acceptance Criteria
- Create/modify/delete/mode/type/symlink changes are each classified correctly on a
  crafted before/after pair (table test).
- A file moved within the workspace is reported as a single `Renamed` (when unambiguous).
- Two identical files both deleted+created stay separate with a caveat (ambiguity).
- Output is deterministic (sorted); `max_files` truncation works.
- Unreadable/special files do not panic.

## Validation
- `cargo test engine::common::scan` with temp before/after trees covering every
  `ChangeKind`, rename, ambiguous rename, truncation.

## Dependencies
08, 11.

## Non-goals
No diff body text (14); no overlay-specific logic (21); no network/blocked capture.

## Design References
DESIGN §6.1 (copy detection), §3.5 (rename best-effort, shadow-path caveat), §6.2
(determinism).
