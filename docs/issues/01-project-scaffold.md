# 01 — Project scaffold, toolchain, lints, module stubs

## Title
Project scaffold, toolchain, lints, and module stubs

## Summary
Create the Rust crate skeleton for `dryrun`: `Cargo.toml`, pinned toolchain, lint
configuration, the module tree from DESIGN §3.2 as compiling stubs, and a `--version`
that works. This is the base every other issue builds on.

## Context
dryrun is a single binary crate with an internal library so tests can reach internals
(DESIGN §3.2). We want strict lints and a pinned toolchain from commit one so later
issues fail fast on style/safety regressions. No product logic here — only structure that
compiles and passes `fmt`/`clippy`.

## Scope
- In: `Cargo.toml`, `rust-toolchain.toml`, `src/main.rs`, `src/lib.rs`, empty-but-
  compiling module files matching DESIGN §3.2, `.gitignore`, a minimal `version` command.
- Out: any engine, CLI flags beyond `version`, config, tests of behavior (later issues).

## Detailed Requirements
1. `Cargo.toml`:
   - `[package]` name `dryrun`, `edition = "2021"` (or newest stable edition), `rust-version`
     set to the chosen MSRV (latest stable minus 2), `license = "MIT OR Apache-2.0"`
     (ADR-006, Accepted 2026-07-11 — confirmed, not tentative), `description`,
     `repository`, `readme = "README.md"`.
   - `[[bin]]` name `dryrun`, `path = "src/main.rs"`.
   - `[lib]` `path = "src/lib.rs"`.
   - `[dependencies]`: add only `clap` (derive), `thiserror`, `anyhow` for now. Other
     crates are added by the issues that need them (research/02 is the allowlist).
   - `[profile.release]`: `opt-level = 3`, `lto = "thin"`, `strip = "symbols"`. Do NOT set
     `panic = "abort"` here — issue 26/36 decides panic strategy with teardown tests.
   - `[lints.rust]` and `[lints.clippy]`: deny `warnings`; deny `clippy::all` and
     `clippy::pedantic` selectively (allow noisy pedantic lints explicitly in-file where
     justified). At minimum: `unsafe_op_in_unsafe_fn = "deny"`, `unused = "deny"`.
2. `rust-toolchain.toml`: `channel = "stable"`, `components = ["rustfmt","clippy"]`.
3. Module stubs under `src/` matching DESIGN §3.2 exactly (cli, config, policy, engine
   incl. common/macos/linux, changes, report, execreal, doctor, error, version). Each is
   an empty `pub mod` or a file with a `//! ` doc comment stating its future role. They
   MUST compile.
4. `src/main.rs`: parse just enough to support `dryrun version [--json]` and route to
   `version::print`. Return `ExitCode`.
5. `src/version.rs`: expose `tool_version()` (from `env!("CARGO_PKG_VERSION")`), `os()`
   (`"macos"`/`"linux"`), and a `print(json: bool)` that prints human or `{"tool_version":
   "..","os":".."}`.
6. `.gitignore`: `/target`, run-dir artifacts, editor files.
7. A top `//!` crate doc in `lib.rs` linking to `docs/DESIGN.md`.

## Acceptance Criteria
- `cargo build` succeeds on macOS and Linux.
- `cargo fmt --check` passes.
- `cargo clippy --all-targets -- -D warnings` passes.
- `dryrun version` prints the version; `dryrun version --json` prints valid JSON with
  `tool_version` and `os`.
- The full module tree from DESIGN §3.2 exists and compiles (verify with `cargo build`).

## Validation
- `cargo build && cargo fmt --check && cargo clippy --all-targets -- -D warnings`
- `cargo run -- version --json | python3 -m json.tool`

## Dependencies
None.

## Non-goals
No sandbox, CLI flags, config, or tests of product behavior. No `panic = "abort"`.

## Design References
DESIGN §3.2 (module layout), §14 (observability, logging TBD), ADR-002 (Rust),
research/02 (dependency policy).
