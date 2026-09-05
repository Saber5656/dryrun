# 11 — Shadow builder and pre-run manifest

## Title
Shadow builder and pre-run manifest

## Summary
Implement the copy/reflink shadow of the workspace used by the macOS and portable engines,
plus the pre-run manifest (path→Meta) that the tree-diff scanner (12) diffs against.

## Context
Copy engines rehearse in a throwaway shadow (DESIGN §6.1 copy engines, §13). Cloning must
be cheap where the FS supports it (APFS `clonefile`, btrfs/XFS reflink) and correct
(fallback plain copy) elsewhere (DESIGN §3.5, research/01).

## Scope
- In: `src/engine/common/shadow.rs` — `build_shadow(policy, run_dir) -> ShadowLayout`, and
  `capture_manifest(root) -> Manifest`.
- Out: macOS `clonefile` FFI (15; this calls it), Landlock/seccomp, scanning (12).

## Detailed Requirements
1. `build_shadow`:
   - Create `run_dir/shadow/` (mode 0700) and clone/copy the workspace subtree into it.
   - Strategy: try reflink (`clonefile` on macOS via 15's FFI; `FICLONE`/`copy_file_range`
     on Linux); on failure or unsupported FS, plain byte copy. Respect `max_disk`/
     `max_files` as a guard (abort → `LimitKind` if the pre-copy itself exceeds budget).
   - Preserve mode, mtime, symlinks (copy symlinks as symlinks, do NOT deref), and
     special files as metadata-only (do not copy device contents).
   - Additional `--writable` roots outside the workspace are each shadowed similarly and
     mapped; record the mapping (original → shadow) for scan + report path translation.
   - Create `run_dir/tmp/` as the private tmp (mode 0700).
2. `capture_manifest(root) -> Manifest`: a deterministic map `relpath -> Meta` for the
   pre-run tree (size, mode, is_symlink, target, blake3 content_hash for regular files
   within a hashing size cap; large files store size only). Persist to
   `run_dir/manifest.json`. This is the "before" state for 12.
3. `ShadowLayout` records: shadow root(s), private tmp, original→shadow path map, and the
   manifest path. Consumed by engines (17/19) and the scanner (12).
4. Path translation helpers: `to_shadow(orig)` and `to_orig(shadow)` so reports show
   original workspace-relative paths even though execution happened in the shadow (DESIGN
   §3.5 absolute-path caveat is documented; here we at least translate the common case).
5. Performance: hashing is parallel (blake3) but bounded; skip hashing files above the cap
   and mark them.

## Acceptance Criteria
- On a reflink-capable FS, shadow creation is fast (reflink path taken — assert via a test
  hook/counter) and file contents match the originals.
- On a non-reflink FS (or forced copy mode), plain copy is used and contents match.
- Symlinks are preserved as symlinks; modes and mtimes preserved.
- `manifest.json` is deterministic for a fixed tree (byte-stable minus any timestamp
  fields) and round-trips via serde.
- Pre-copy exceeding `max_disk`/`max_files` aborts with the right `LimitKind`.

## Validation
- `cargo test engine::common::shadow` with temp trees (regular files, subdirs, symlinks,
  a large file), asserting content equality, symlink preservation, manifest determinism,
  and the reflink-vs-copy path selection via an injected FS-capability flag.

## Dependencies
05, 06, 15. (Uses 15's `clonefile` FFI on the macOS reflink path, `cfg(target_os)`-gated;
the plain-copy path needs only 05/06, so 11 can begin before 15 lands but the macOS reflink
fast-path is wired once 15 is available.)

## Non-goals
No change classification (12); no sandbox enforcement.

## Design References
DESIGN §3.5 (matrix, caveat), §6.1 (copy detection), §13 (run dir), research/01
(clonefile/reflink), ADR-004.
