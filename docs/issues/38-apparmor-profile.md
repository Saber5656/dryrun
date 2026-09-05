# 38 — AppArmor userns profile for the overlay engine on restricted distros

## Title
AppArmor userns profile for the overlay engine on restricted distros

## Summary
Author, validate, package, and document the AppArmor profile that lets the installed
`dryrun` binary create unprivileged user namespaces on distros that restrict them by
default (Ubuntu 23.10+/24.04), which is the remediation `dryrun doctor` already points users
to (research/01, DESIGN §3.4).

## Context
Ubuntu 24.04 restricts unprivileged user namespaces via AppArmor by default, disabling the
`linux-overlay` engine and forcing the `linux-portable` fallback (research/01). Issue 27
(`doctor`) and issue 33/36 reference "install the shipped AppArmor profile" as the fix, but
no issue owned creating it. This issue owns that artifact end to end so the remediation is
real, not vaporware.

## Scope
- In: an AppArmor profile file under `packaging/apparmor/` granting `userns,` to the dryrun
  binary path, an install script/doc, CI validation that it loads and works, and packaging
  wiring so releases ship it.
- Out: the doctor text (27, references this), the release tarball assembly (36, packages
  this), the engine code (20/21).

## Detailed Requirements
1. Create `packaging/apparmor/dryrun` (or `usr.local.bin.dryrun`): a minimal AppArmor
   profile that attaches to the installed binary path(s) and includes the `userns,` rule so
   the confined `dryrun` may create user namespaces, while otherwise remaining unconfined
   enough not to break normal operation. Document the assumed install path(s) and how to
   adjust (`/usr/bin/dryrun`, `/usr/local/bin/dryrun`, `~/.cargo/bin/dryrun`).
2. Install documentation (`packaging/apparmor/README.md`): the exact steps —
   `sudo cp packaging/apparmor/dryrun /etc/apparmor.d/ && sudo apparmor_parser -r
   /etc/apparmor.d/dryrun` — plus how to verify (`aa-status`), how to adjust the path, and
   the admin-alternative (`sysctl kernel.apparmor_restrict_unprivileged_userns=0`, noting
   its system-wide security trade-off). Make clear this is optional; without it dryrun still
   works via the portable engine.
3. CI validation (extends 33's restricted job or a dedicated job): on a distro that
   restricts userns, assert that (a) without the profile the overlay engine probes unusable
   and doctor recommends this profile, and (b) after installing the profile, the overlay
   engine becomes usable and its integration tests pass. Guard so this runs only where
   AppArmor + restriction are present.
4. Release packaging (coordinate with 36): include `packaging/apparmor/dryrun` and its
   README in the Linux release tarball.
5. Security note (coordinate with 35): document in `SECURITY.md` that installing the profile
   grants the dryrun binary the ability to create user namespaces, and what that does/does
   not expose.

## Acceptance Criteria
- `packaging/apparmor/dryrun` exists and `apparmor_parser` loads it without error on a
  supported distro.
- On a userns-restricted host: overlay is unusable before install and usable after, verified
  by `dryrun doctor` and an overlay integration test in CI.
- Install/verify/adjust steps are documented and accurate.
- The profile is packaged into the Linux release artifact (36).

## Validation
- CI job (33/38) on a userns-restricted image: probe before/after profile install; run one
  overlay integration test after install. `apparmor_parser -r` succeeds.

## Dependencies
01 (repo), 20/21 (overlay engine to validate against), and wires into 27, 33, 36.

## Non-goals
No bypasses of AppArmor; no automatic installation by dryrun (the user/admin installs it;
agents/tools do not modify system security config — project rule). No non-AppArmor LSM
(SELinux) profile in v1.

## Design References
DESIGN §3.4 (remediation), §14 (doctor), research/01 (Ubuntu userns restriction), ADR-001.
