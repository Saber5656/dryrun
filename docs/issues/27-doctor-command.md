# 27 — `dryrun doctor` capability report

## Title
`dryrun doctor` capability report

## Summary
Implement `dryrun doctor [--json]`: probe the environment and print which engines are
usable/degraded/blocked with concrete remediation (especially the Ubuntu 24.04 userns case),
plus OS/kernel/APFS/Landlock-ABI facts (DESIGN §14, research/01).

## Context
Because engine availability varies (Ubuntu userns block, non-APFS, old kernels), users need
a first-class way to understand and fix their environment (DESIGN §3.4 remediation, §14).

## Scope
- In: `src/doctor/mod.rs` — gather probes (07) + environment facts, render human + JSON.
- Out: engine selection logic (07 provides probes); it does not run a command.

## Detailed Requirements
1. Collect: OS + version (macOS product version / Linux kernel release), arch; per-engine
   `Probe` (usable/degraded/reasons) from 07; macOS APFS status of cwd volume; Linux userns
   permission status (attempt the non-mutating probe), AppArmor restriction detection,
   Landlock presence + ABI level, overlayfs availability, reflink capability of cwd fs.
2. Human output: a checklist with ✓/⚠/✗ per engine and per capability, and for each blocker
   a **remediation** line. For Ubuntu-userns-blocked, name the concrete options from
   research/01: install the shipped AppArmor profile (owned by issue 38, packaged by 36 —
   point at `packaging/apparmor/README.md`), or the admin sysctl
   `kernel.apparmor_restrict_unprivileged_userns=0`, and state which engine is used instead
   (portable). Do not promise a profile that issue 38 has not produced; keep this text in
   sync with 38.
3. `--json`: a stable object (`schema_version`, `os`, `engines: [...]`, `capabilities:
   {...}`) suitable for automation; snapshot-tested with volatile fields normalized.
4. Exit code: 0 if at least one engine is usable; a non-zero (41) if none is usable
   (mirrors selection).
5. `doctor` must not create sandboxes, mount anything persistent, or modify system state.

## Acceptance Criteria
- On a normal Linux dev box, doctor reports overlay usable (or portable if userns blocked)
  and prints accurate kernel/Landlock facts.
- On a userns-restricted host, doctor shows overlay ✗ with the exact remediation and
  portable ✓.
- On macOS, doctor shows seatbelt ✓ and APFS status (degraded note if non-APFS).
- `--json` validates against its documented shape and is snapshot-stable.
- No system state is modified (verified by the probes being read-only).

## Validation
- `cargo test doctor::` (unit with injected probe results for each scenario + JSON
  snapshot). Real-environment checks run in CI (33) on each OS/config.

## Dependencies
07.

## Non-goals
No fixing of the environment (only guidance); no command rehearsal.

## Design References
DESIGN §3.4 (remediation), §14 (doctor fields), research/01 (Ubuntu userns), ADR-001.
