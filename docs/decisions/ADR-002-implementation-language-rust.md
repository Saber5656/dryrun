# ADR-002: Implementation language — Rust

- Status: Accepted
- Date: 2026-07-10
- Deciders: user (product owner), Fable (design)
- Related: [research/02-rust-dependencies.md](../research/02-rust-dependencies.md)

## Context

dryrun needs: direct calls into low-level OS sandbox APIs (`sandbox_init`, `clonefile`,
`unshare`/`mount`, Landlock, seccomp), a single self-contained binary for easy OSS
distribution, memory safety in a security tool, and enough type-system strength that a
lower-capability implementation agent is caught by the compiler rather than by review.
The user selected **Rust** during clarification.

## Decision

Implement dryrun in Rust (stable toolchain, edition pinned via `rust-toolchain.toml`).

- FFI to macOS Seatbelt/`clonefile` via a small hand-written `extern "C"` surface.
- Linux syscalls via `nix` (fork/exec/unshare/mount/signals) + the `landlock` and
  `seccompiler` crates.
- Strict lints (`clippy -D warnings`), `rustfmt`, `cargo-deny` for licenses + RustSec
  advisories, `--locked` builds.

Dependency policy and candidate crates are in
[research/02-rust-dependencies.md](../research/02-rust-dependencies.md).

## Consequences

Positive:

- Single binary per (os, arch); trivial `cargo install` and GitHub Release distribution.
- Ownership/borrow checking eliminates a class of bugs in the child-process/pipe/teardown
  code that a weaker agent would otherwise get wrong.
- Mature crates for every subsystem; `unsafe` confined to named OS-boundary modules with
  mandatory `// SAFETY:` comments.

Negative / accepted costs:

- macOS Seatbelt and `clonefile` need hand-written FFI (small, well-scoped).
- Contributors need a Rust toolchain; mitigated by pinned toolchain + documented setup.
- Longer compile times than Go; acceptable for a CLI of this size.

## Alternatives considered

- **Go**: faster to write, rich namespace tooling, but macOS Seatbelt requires cgo FFI
  and the single-binary security story (no GC pauses during teardown, tighter unsafe
  surface) is weaker for a security tool. Rejected in favor of Rust's stronger
  compile-time guarantees for the sensitive core.
