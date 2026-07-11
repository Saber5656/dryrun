# Research: OS-native sandbox primitives for dryrun

Date: 2026-07-10
Status: informs ADR-001 (platform & isolation strategy)

dryrun rehearses a command inside an OS-native sandbox (no container runtime), records
filesystem changes scoped to a workspace, and blocks network access by default. This
document records the primitives available on each target OS, their constraints, and the
facts verified by web research that materially shaped the design.

## Summary of conclusions

| Concern | macOS | Linux (primary) | Linux (fallback) |
|---|---|---|---|
| Write capture | APFS `clonefile(2)` shadow copy of workspace; command runs in shadow | overlayfs upperdir over the workspace path inside a mount namespace | reflink/plain shadow copy, command runs in shadow |
| Write confinement outside workspace | Seatbelt profile: `deny file-write*` except shadow + private tmp | Landlock ruleset (write rights only on overlay workspace + private tmp) | Landlock ruleset |
| Network default-deny | Seatbelt `deny network*` | dedicated network namespace with only loopback | seccomp filter denying `AF_INET`/`AF_INET6`/`AF_PACKET` socket creation |
| Privileges required | none (unprivileged APIs) | unprivileged user namespace must be permitted by the distro | none beyond `no_new_privs` |
| Minimum OS level | macOS 13+ on APFS (older/non-APFS falls back to copy) | kernel >= 5.13 (overlayfs-in-userns >= 5.11, Landlock >= 5.13) | kernel >= 5.13 |

## macOS: Seatbelt (`sandbox_init` / `sandbox-exec`) + APFS `clonefile`

Verified facts (2026-07):

- `sandbox-exec(1)` and the `sandbox_init(3)` C API are **deprecated but still functional
  through macOS 26 (Tahoe)**. The deprecation has been in place for years with no removal
  and no published replacement API that can apply an SBPL profile to a child process
  without App Sandbox entitlements. Major agent CLIs (Claude Code, OpenAI Codex) rely on
  Seatbelt today.
- On macOS 15 (Sequoia) the `sandbox-exec` CLI prints a deprecation warning to stderr on
  every invocation. Calling `sandbox_init()` from our own child process avoids depending
  on the CLI binary and its warning.
- **macOS 26 (Tahoe) regression risk for deny-default profiles**: processes such as
  `zsh 5.9` now read additional sysctls (`hw.targettype`, `hw.osenvironment`,
  `hw.features.allows_security_research`, `hw.jetsam_properties_product_type`, ...) which
  a `(deny default)` profile rejects, breaking previously-working profiles.

Design consequences:

- Use an **allow-default SBPL profile with targeted denies** (`deny file-write*` outside
  the allowlist, `deny network*` unless `--allow-network`). This matches dryrun's threat
  model (reads are permitted by design; writes and network are the controlled surfaces)
  and is robust against Tahoe-style "process reads one more sysctl" regressions.
- Apply the profile via `sandbox_init()` in the forked child before `execvp`, not via the
  `sandbox-exec` binary.
- Treat possible future removal of Seatbelt as a **known unknown** in the risk register;
  the engine abstraction must allow replacing the macOS engine without touching the core.
- APFS `clonefile(2)` provides constant-time copy-on-write clones of files; cloning a
  workspace tree is metadata-cost only. Clones require **same-volume** source/target;
  non-APFS or cross-volume setups must fall back to a plain copy with a size guard.
- Because the rehearsal runs in a shadow directory at a different absolute path,
  absolute-path writes into the *original* workspace path are denied by the profile and
  surface as violations instead of silently diverging. This asymmetry vs Linux is a
  documented platform caveat.

Sources:

- [apple/containerization#737 — Clarify `sandbox-exec` deprecation timeline](https://github.com/apple/containerization/issues/737)
- [sandbox-exec(1) man page](https://manp.gs/mac/1/sandbox-exec)
- [anthropics/claude-code#49820 — Tahoe: zsh reads hw.* sysctls not whitelisted under deny-default](https://github.com/anthropics/claude-code/issues/49820)
- [anthropics/claude-code#26095 — Sandbox failed to initialize on macOS 26 Tahoe](https://github.com/anthropics/claude-code/issues/26095)
- [openai/codex#215 — sandbox-exec was deprecated on macOS](https://github.com/openai/codex/issues/215)
- [macOS's little-known command-line sandboxing tool (HN discussion, 2025)](https://news.ycombinator.com/item?id=47101200)
- [Run code in a macOS Sandbox (myByways)](https://mybyways.com/blog/run-code-in-a-macos-sandbox)
- [macOS: App sandboxing via sandbox-exec (Karl Tarvas)](https://www.karltarvas.com/macos-app-sandboxing-via-sandbox-exec/)

## Linux primary engine: user namespaces + overlayfs + Landlock + netns

Verified facts (2026-07):

- **overlayfs can be mounted by root inside an unprivileged user namespace since kernel
  5.11.** Overlayfs parses no user-supplied data beyond pathnames, which is the upstream
  rationale for allowing it. Apptainer/Singularity rely on this in production.
- **Ubuntu 23.10+ restricts unprivileged user namespaces via AppArmor, enabled by default
  in Ubuntu 24.04 LTS.** Unconfined unprivileged processes may create a user namespace but
  are denied capabilities inside it (so `mount(2)` fails), unless an AppArmor profile with
  a `userns,` rule applies, the binary runs via a permitted profile, or the sysctl
  `kernel.apparmor_restrict_unprivileged_userns` is relaxed. Several bypasses exist
  (aa-exec/busybox routes), but dryrun must not rely on bypasses.
- Landlock (LSM, kernel >= 5.13) lets an unprivileged process restrict **its own**
  filesystem access by path. ABI has grown incrementally: v1 (5.13) filesystem
  read/write/exec rights; v2 (5.19) `LANDLOCK_ACCESS_FS_REFER` controlling
  rename/hardlink across domain boundaries; v3 (6.2) truncate; v4 (6.7) TCP bind/connect
  restrictions; v5 (6.10) ioctl on devices; v6 (6.12) scoped signals/abstract sockets.
  (Verify exact ABI availability at implementation time via `landlock_create_ruleset`
  probe rather than kernel version parsing.)

Design consequences:

- Primary engine (`linux-overlay`): unshare user+mount+pid+net namespaces; mount
  overlayfs **at the workspace path itself** (lowerdir=workspace, upper/work in the run
  directory, `userxattr` option required for user-namespace mounts) so absolute paths
  keep working; mount a private tmpfs on `/tmp`; bring up only loopback in the netns.
- Overlayfs only captures writes **under the workspace mount point**. Writes elsewhere in
  the filesystem would hit the real files, so the engine MUST also apply a Landlock
  ruleset granting write rights only on the (overlaid) workspace and the private tmp.
  Landlock is therefore a hard requirement of the primary engine, giving kernel >= 5.13
  as the effective floor.
- The upperdir *is* the changeset: files present in upper are created/modified; character
  devices 0:0 are whiteouts (deletions); opaque directories are marked with
  `user.overlay.opaque` xattr under `userxattr` mounts.
- Because Ubuntu 24.04 (a top-2 distro) blocks the primary engine by default, v1 MUST
  ship: (a) a capability probe that detects this exact condition and explains remediation
  (install the shipped AppArmor profile, or admin opt-out), (b) `dryrun doctor` surfacing
  it, and (c) an automatic fallback engine that works without user namespaces.

Sources:

- [overlayfs update for 5.11 (unprivileged mounts) — LKML pull request](https://lkml.kernel.org/linux-fsdevel/160823647214.7820.4991320145990086247.pr-tracker-bot@kernel.org/T/)
- [Apptainer admin guide — user namespaces & unprivileged overlay (kernel >= 5.11)](https://apptainer.org/docs/admin/1.3/user_namespace.html)
- [Ubuntu blog — Restricted unprivileged user namespaces in Ubuntu 23.10](https://ubuntu.com/blog/ubuntu-23-10-restricted-unprivileged-user-namespaces)
- [Ubuntu 24.04 LTS release notes](https://documentation.ubuntu.com/release-notes/24.04/)
- [Ubuntu discourse — Understanding AppArmor user namespace restriction](https://discourse.ubuntu.com/t/understanding-apparmor-user-namespace-restriction/58007)
- [Qualys — Three bypasses of Ubuntu's unprivileged user namespace restrictions](https://www.qualys.com/2025/three-bypasses-of-Ubuntu-unprivileged-user-namespace-restrictions.txt)
- [DEVCORE — Bypassing Ubuntu's unprivileged namespace restriction](https://devco.re/blog/2025/06/26/the-journey-of-bypassing-ubuntus-unprivileged-namespace-restriction-en/)
- [Codex sandbox error on Ubuntu 24.04: the AppArmor fix](https://www.jdhodges.com/blog/codex-sandbox-ubuntu-24-04-fix/)

## Linux fallback engine: shadow copy + Landlock + seccomp

No verified-web facts needed beyond the above; standard kernel features:

- Shadow copy of the workspace into the run directory. Use `copy_file_range`/reflink
  (`FICLONE`) on btrfs/XFS for cheap copies; plain copy elsewhere, guarded by a size
  limit and file-count limit from the policy.
- Landlock confines writes to the shadow + private tmp (same ruleset builder as the
  primary engine). No namespaces required, so it works under Ubuntu's AppArmor
  restriction and in locked-down CI.
- Network default-deny cannot use a netns (requires userns), so install a seccomp filter
  denying `socket(2)` for `AF_INET`, `AF_INET6`, and `AF_PACKET` (allow `AF_UNIX`,
  `AF_NETLINK` for local IPC). Requires `no_new_privs`, which the engine sets anyway.
- Same absolute-path caveat as macOS (rehearsal runs at a shadow path); documented
  identically.

## Options considered and rejected

| Option | Why rejected |
|---|---|
| Container runtime (Docker/Podman/OrbStack) | External dependency; rehearses inside a different OS image, destroying "what happens to *my* machine" fidelity (user decision Q2) |
| ptrace/syscall-interposition capture | Order-of-magnitude slowdown, fragile across syscall surface, complex security review |
| `LD_PRELOAD`/`DYLD_INSERT_LIBRARIES` interposition | Trivially bypassed by static binaries; not a security boundary |
| macOS Endpoint Security framework | Requires entitlements + notarization approval; unsuitable for an OSS CLI v1 |
| FUSE overlay on macOS (macFUSE) | Third-party kext/system extension dependency contradicts "no external runtime" |
| Whole-system shadow (rehearse writes anywhere) | On macOS requires heavyweight tech above; asymmetric guarantees rejected in requirements (workspace-scoped chosen) |
