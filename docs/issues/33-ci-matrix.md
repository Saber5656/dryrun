# 33 — CI matrix (macOS + Linux userns on/off)

## Title
CI matrix (macOS + Linux userns on/off)

## Summary
Set up GitHub Actions CI running fmt, clippy, build, and the full test suite on macOS (APFS)
and Linux in **two** configurations — user namespaces enabled (overlay engine) and
restricted (portable fallback) — so both Linux engines are exercised on every PR
(DESIGN §15, research/01).

## Context
The three engines only get real coverage on their target environments. The
userns-enabled/restricted split is essential because Ubuntu 24.04 (a top distro) blocks the
overlay engine by default; the portable fallback must be proven on every PR.

## Scope
- In: `.github/workflows/ci.yml` (or equivalent), job matrix, engine-gated test wiring.
- Out: release workflow (36); cargo-deny config (32, invoked here).

## Detailed Requirements
1. Jobs:
   - `lint`: `cargo fmt --check`, `cargo clippy --all-targets -- -D warnings`, `cargo deny
     check` (32).
   - `test-macos`: on a macOS runner (APFS), `cargo test --locked` incl. os-gated seatbelt
     engine + security suites (28–31) that apply.
   - `test-linux-userns`: on a Linux runner with unprivileged userns ENABLED; runs overlay
     engine tests (20/21) + portable + security suites.
   - `test-linux-restricted`: Linux runner with unprivileged userns RESTRICTED (set
     `kernel.apparmor_restrict_unprivileged_userns=1` or use a distro/image that restricts,
     or drop the capability) so overlay is unavailable and the **portable** engine +
     fallback selection (07) are exercised. Assert `doctor` reports overlay ✗/portable ✓.
     Issue 38's AppArmor-profile validation (overlay usable after installing the profile)
     extends this job or runs as a sibling job.
2. Cache cargo registry/target for speed; pin toolchain via `rust-toolchain.toml`.
3. Fail the build if any os-gated engine test is *skipped when it should run* (guard against
   accidental universal skips) — e.g. the userns-enabled job must actually run overlay
   tests, not skip them. Use an env flag the tests read to assert "this engine must be
   tested here."
4. Upload test logs/artifacts on failure for debugging.
5. Run on PRs and pushes to the default branch; required for merge (documented; branch
   protection is configured by the user, not the agent).

## Acceptance Criteria
- All jobs green on a correct tree.
- The userns-enabled job runs overlay tests (not skipped); the restricted job runs portable
  and asserts overlay is unavailable.
- macOS job runs seatbelt tests.
- A clippy warning or `cargo deny` violation fails CI.

## Validation
- CI passes on the design/implementation PRs; verify each job's log shows the intended
  engine tests actually executed (the "must-test" guard).

## Dependencies
26 (something to test end-to-end), 32 (deny gate). Engine tests come from 17/19/21/28–31.

## Non-goals
No release/publish (36); no branch-protection setup (user-managed).

## Design References
DESIGN §15 (testing/CI matrix), §3.4 (fallback), research/01 (Ubuntu userns), ADR-001.
