# 32 — cargo-deny and supply-chain gates

## Title
cargo-deny and supply-chain gates

## Summary
Add `deny.toml` and a CI gate enforcing the dependency policy from research/02: license
allowlist, RustSec advisory checks, banned/duplicate crates, and `--locked` builds with a
committed `Cargo.lock` (DESIGN §10.6).

## Context
Every dependency is supply-chain attack surface (DESIGN §10.6). Automated gates keep the
dependency set clean and auditable as the tree grows across issues.

## Scope
- In: `deny.toml`, `Cargo.lock` committed, a CI job/step running `cargo deny check`.
- Out: the rest of CI (33); release signing/checksums (36).

## Detailed Requirements
1. `deny.toml` sections:
   - `[licenses]`: allow `MIT`, `Apache-2.0`, `Apache-2.0 WITH LLVM-exception`, `BSD-2-
     Clause`, `BSD-3-Clause`, `Unicode-3.0`/`Unicode-DFS-2016` (as needed by deps),
     `ISC`, `Zlib`. Deny copyleft that conflicts with the project license (GPL/AGPL) unless
     explicitly allowed. `confidence-threshold` set high; `unlicensed = "deny"`.
   - `[advisories]`: use the RustSec DB; `vulnerability = "deny"`, `unmaintained = "warn"`
     (or deny with explicit allowlist), `yanked = "deny"`.
   - `[bans]`: `multiple-versions = "warn"` (deny where feasible), `wildcards = "deny"`;
     deny any crate not needed (optional explicit deny list for known-bad).
   - `[sources]`: only crates.io + the project git; `unknown-registry = "deny"`.
2. Commit `Cargo.lock`; CI builds/tests with `--locked` (fail if lock is stale).
3. CI step: `cargo deny check advisories bans licenses sources` fails the build on
   violations. Pin the `cargo-deny` version used in CI.
4. Document in `CONTRIBUTING.md` (35) that new deps must pass `cargo deny` and follow
   research/02 policy (justify each addition, keep `unsafe` in named modules).
5. Add a `cargo audit`-equivalent only if not redundant with `cargo deny advisories`
   (avoid duplicate tooling).

## Acceptance Criteria
- `cargo deny check` passes locally and in CI with the current tree.
- Introducing a GPL-only or yanked dependency fails the check (verified once with a
  temporary edit, then reverted).
- CI builds use `--locked`; a deliberately stale lock fails CI.

## Validation
- `cargo deny check` in CI on every PR; `cargo build --locked` / `cargo test --locked`.

## Dependencies
01.

## Non-goals
No release artifacts/signing (36); no test content (28–31).

## Design References
DESIGN §10.6 (supply chain), research/02 (dependency policy), ADR-002.
