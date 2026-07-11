# 15 — macOS Seatbelt/clonefile FFI

## Title
macOS Seatbelt and clonefile FFI bindings

## Summary
Hand-write the minimal `extern "C"` bindings for macOS `sandbox_init`/`sandbox_free_error`
and `clonefile`, with safe Rust wrappers and `// SAFETY:` documentation, isolated in one
module (DESIGN §3.2, ADR-001/002).

## Context
No maintained crate cleanly applies an SBPL profile to the current process without App
Sandbox entitlements, and the API is deprecated-but-functional (research/01). We keep a
tiny, audited FFI surface rather than depend on abandoned wrappers.

## Scope
- In: `src/engine/macos/ffi.rs` — `extern "C"` decls + safe wrappers `sandbox_apply(profile:
  &CStr) -> Result<(), String>` and `clonefile(src, dst) -> io::Result<()>`.
- Out: profile text (16), engine orchestration (17), shadow logic (11 calls `clonefile`).

## Detailed Requirements
1. Declare `extern "C"`:
   - `fn sandbox_init(profile: *const c_char, flags: u64, errorbuf: *mut *mut c_char) ->
     c_int;`
   - `fn sandbox_free_error(errorbuf: *mut c_char);`
   - `fn clonefile(src: *const c_char, dst: *const c_char, flags: u32) -> c_int;`
   Link against the system libraries (libSystem provides these; document the linkage).
2. Safe wrapper `sandbox_apply(profile: &CStr) -> Result<(), String>`:
   - calls `sandbox_init(profile, SANDBOX_STRING_PROFILE_FLAG(0), &mut err)`; on non-zero,
     copies the error string, frees it with `sandbox_free_error`, returns `Err`.
   - MUST be called in the child (post-fork, pre-exec) — document this; the function itself
     just applies to the calling process.
   - Every `unsafe` block carries a `// SAFETY:` note (pointer validity, ownership of the
     error buffer, no use-after-free).
3. Safe wrapper `clonefile(src, dst) -> io::Result<()>` mapping errno; caller handles
   `ENOTSUP`/cross-device by falling back to copy (11).
4. Suppress/avoid the deprecation warning at build (the symbol is still linkable);
   document that we intentionally use a deprecated-but-available API and reference
   research/01 + ADR-001 U3.
5. `#![cfg(target_os = "macos")]`-gate the whole module so non-macOS builds exclude it.
6. Provide a tiny self-test wrapper used by 17's tests (e.g. apply an allow-all profile
   and confirm success; apply a malformed profile and confirm a clean `Err`).

## Acceptance Criteria
- On macOS, applying a syntactically valid trivial profile returns `Ok`; a malformed
  profile returns `Err` with the libsandbox message and no leak/crash.
- `clonefile` succeeds on APFS same-volume; returns a mappable error on unsupported FS so
  the caller can fall back.
- Module compiles only on macOS; Linux build excludes it.
- All `unsafe` has `// SAFETY:` comments; `cargo clippy` clean.

## Validation
- `cargo test --target-os macos engine::macos::ffi` (os-gated) for valid/invalid profile
  and clonefile success/fallback.

## Dependencies
01.

## Non-goals
No profile authoring (16); no process spawning (17).

## Design References
DESIGN §3.2 (ffi module), §10.3 (FFI boundary), §2.4 U3, research/01, ADR-001/002.
