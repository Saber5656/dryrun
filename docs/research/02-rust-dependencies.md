# Research: Rust dependency policy and candidate crates

Date: 2026-07-10
Status: informs ADR-002 (implementation language) and issue 01 (scaffold)

dryrun is a security-sensitive tool: every dependency is attack surface (supply chain)
and audit surface. This document sets the dependency policy and lists the candidate
crates per subsystem. Exact versions are chosen at implementation time (pin via
`Cargo.lock`, enforce via `cargo deny`); the names below were current as of the design
date.

## Dependency policy

1. Prefer the standard library; add a crate only when it removes meaningful risk or
   effort (parsing, OS API bindings, diffing).
2. Every crate must be: actively maintained, widely used (high download count / used by
   major projects), and license-compatible (MIT/Apache-2.0/BSD).
3. `deny.toml` (cargo-deny) gates: license allowlist, advisory database (RustSec), no
   duplicate major versions where avoidable, no wildcard versions.
4. Build with `--locked` in CI and release; commit `Cargo.lock`.
5. No crates that execute network I/O at build time; no proc-macro crates beyond the
   mainstream ones pulled by serde/clap/thiserror.
6. `unsafe` is expected only in the OS boundary modules (FFI to `sandbox_init`,
   `clonefile`, namespace/landlock/seccomp syscalls); all `unsafe` blocks require a
   `// SAFETY:` comment and are concentrated in the engine OS modules per DESIGN §3.2:
   `src/engine/macos/ffi.rs` and `src/engine/linux/{namespace,landlock,seccomp}.rs`.

## Candidate crates by subsystem

| Subsystem | Crate | Role | Notes |
|---|---|---|---|
| CLI parsing | `clap` (derive) | flags/subcommands/help | de-facto standard; `clap_mangen` + `clap_complete` for man page/completions in the docs issue |
| Errors | `thiserror` + `anyhow` | typed internal errors / top-level context | typed errors at module boundaries, `anyhow` only in `main`/orchestrator |
| Serialization | `serde`, `serde_json`, `toml` | JSON report, config file | `deny_unknown_fields` on config structs |
| Unix APIs | `nix` (or `rustix`) | fork/exec, pipes, signals, mounts, unshare | choose one at scaffold time; do not mix; both are mainstream. `nix` chosen as default candidate for breadth (mount/unshare coverage) |
| Landlock | `landlock` crate | ruleset builder | maintained by the Landlock kernel author; probe ABI at runtime |
| seccomp | `seccompiler` | socket-family deny filter for the portable engine | small, no libseccomp C dependency |
| macOS FFI | hand-written `extern "C"` | `sandbox_init`, `sandbox_free_error`, `clonefile` | tiny surface; avoids abandoned wrapper crates; lives in `src/engine/macos/ffi.rs` |
| Text diff | `similar` | unified diffs for text files | powers `insta`; battle-tested |
| Tree walking | `walkdir` | scanner tree traversal | deterministic ordering option needed for stable reports |
| Hashing | `blake3` | content-change detection, diff dedup keys | fast, parallel |
| Terminal | `anstream`/`anstyle` (clap family) or `owo-colors` | color handling incl. NO_COLOR | pick within clap's `anstream` family to avoid duplicates |
| Temp files | `tempfile` | run-dir staging, tests | |
| Time | `std::time` + `jiff` or `time` | timestamps in reports | smallest adequate option; RFC3339 output |
| Test snapshots | `insta` (dev-dep) | golden tests for reports/help/SBPL profiles | dev-only |
| Property tests | `proptest` (dev-dep) | changeset scanner edge cases | dev-only |

Explicitly avoided:

| Crate/approach | Reason |
|---|---|
| `tokio`/async runtime | The tool is a sequential pipeline around one child process; threads suffice (capture tee, disk monitor). Async adds audit surface without benefit |
| `libc`-level hand-rolled everything | `nix`/`rustix` are safer wrappers with the same footprint |
| heavyweight TUI (`ratatui`) in v1 | v1 renders plain colored text + pager; TUI is a v2 idea |
| any telemetry/update-check crate | dryrun performs zero phone-home by policy (see DESIGN security section) |

## Toolchain

- Stable Rust, pinned via `rust-toolchain.toml` (edition 2021 or the newest stable
  edition at scaffold time).
- `cargo fmt --check`, `cargo clippy --all-targets -- -D warnings` gate CI.
- MSRV: latest stable minus 2 at scaffold time; recorded in `Cargo.toml` `rust-version`.
- Release builds: `--locked --release`, `strip = "symbols"`, `panic = "abort"` evaluated
  at release-pipeline issue (panic strategy must not skip sandbox teardown; decide there
  with tests).
