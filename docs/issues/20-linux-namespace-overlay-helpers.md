# 20 — Linux namespace and overlayfs helpers

## Title
Linux namespace and overlayfs helpers

## Summary
Implement the unprivileged user+mount+pid+net namespace setup and the overlayfs mount
(lower=workspace, upper/work in the run dir, `userxattr`) plus loopback-only networking,
the primitives the overlay engine (21) composes (research/01).

## Context
The overlay engine's isolation depends on correctly entering namespaces and mounting
overlayfs under `userxattr` (required for userns mounts). This is the trickiest OS code;
isolating it here keeps 21 focused on change scanning + lifecycle.

## Scope
- In: `src/engine/linux/namespace.rs` — `unshare_all()`, uid/gid map setup, `mount_overlay()`,
  `mount_private_tmp()`, `bring_up_loopback()`, `mount_proc()`.
- Out: Landlock (13), seccomp (18), the engine impl + scanning (21).

## Detailed Requirements
1. `unshare_all(network: NetworkPolicy)`: create user+mount+net(when Deny) namespaces and
   **request** a new pid namespace. Write `uid_map`/`gid_map` (map current uid→0 inside)
   and `setgroups=deny` as required for unprivileged userns. On EPERM (AppArmor-restricted),
   return a typed error so the engine's probe/selection reports unusable (07 fallback to
   portable).
   - **PID-namespace correctness (critical)**: `unshare(CLONE_NEWPID)` does NOT move the
     calling process into the new pid namespace; it only causes the caller's *next* child to
     become PID 1 there. Therefore the sequence is: (a) set up user, mount, and net
     namespaces in the current process; (b) `unshare(CLONE_NEWPID)`; (c) `fork()` — the
     child is PID 1 (the namespace "init"); (d) the PID-1 child mounts a fresh `/proc`
     (step 5), performs the overlay/tmp mounts (steps 2–3), applies Landlock/seccomp, then
     `fork/exec`s the actual command and **reaps** it plus any orphaned descendants (acts as
     init: `waitpid` loop until the command's pid exits, then propagate its status). Provide
     a helper `spawn_pid1(closure)` that encapsulates this and returns the command's exit
     status to the parent (outside the namespaces) via a pipe. Document that the monitor (10)
     signals the PID-1 child's process group to tear down the whole tree.
2. `mount_overlay(workspace, run_dir) -> merged_path`: overlayfs cannot use a directory as
   both its own mountpoint and its `lowerdir` reliably. Use this concrete sequence inside
   the mount namespace (after making the root mount `MS_PRIVATE|MS_REC` to stop
   propagation):
   - bind-mount the real `<workspace>` read-only to a private lower path
     `<run_dir>/overlay/lower` (`mount --bind` then remount `MS_RDONLY`), so the lower tree
     is stable and independent of the mountpoint,
   - create `<run_dir>/overlay/upper` and `<run_dir>/overlay/work` (must be on the same
     filesystem as each other and NOT inside the lower tree),
   - `mount -t overlay overlay -o lowerdir=<run_dir>/overlay/lower,upperdir=<run_dir>/overlay/
     upper,workdir=<run_dir>/overlay/work,userxattr <workspace>` — mounting the merged view
     **over the original workspace path** so absolute paths keep working. If mounting over
     the original path is not possible, mount at `<run_dir>/overlay/merged` and `chdir`
     there, adding the `shadow path` caveat.
   - all mounts private (`MS_PRIVATE`) to avoid leaking into the host mount table.
   Return the merged path (normally `<workspace>`).
3. `mount_private_tmp(run_dir)`: tmpfs (or bind of `run_dir/tmp`) at `/tmp` inside the mount
   ns; set `TMPDIR`.
4. `bring_up_loopback()`: in the net namespace, set `lo` up so localhost works while
   external network is absent by construction.
5. `mount_proc()`: called from the PID-1 child (step 1c), mount a fresh `proc` at `/proc`
   in the new pid namespace so tools relying on `/proc` see the namespaced pids and reaping
   is correct. Mounting `/proc` from a process that is NOT pid 1 of the namespace fails or
   shows the wrong pids — hence it must run in the `spawn_pid1` child.
6. All mounts/unshares via `nix`; every `unsafe`/raw syscall has `// SAFETY:`; all failures
   are typed and fail-closed. Provide teardown helpers (unmount in the right order) usable
   by 21's cleanup, though namespace teardown is mostly automatic on process exit.
7. Record which namespaces were entered for the report/doctor.

## Acceptance Criteria
Environment: Linux, user namespaces enabled.
- `unshare_all` succeeds on a userns-enabled kernel and returns a typed EPERM error on a
  userns-restricted one (both asserted; the latter drives fallback).
- `spawn_pid1` produces a child that is PID 1 inside the namespace (assert `getpid()==1` in
  the child) and correctly reaps a grandchild/orphan (spawn a backgrounded process; assert
  no zombie/orphan survives the child's exit).
- Overlay mount via the bind-RO-lower → overlay-over-workspace sequence succeeds with
  `userxattr`; a file created inside appears in `upper`, and the real workspace (the bind
  source) is byte-unchanged after unmount.
- Loopback is up in the netns; no external interface exists.
- `/proc` (mounted from the PID-1 child) shows namespaced pids; private `/tmp` is mounted.

## Validation
- `cargo test --test namespace_it` (os+userns gated; skipped with a clear message when
  userns is blocked). CI (33) runs it on a userns-enabled job.

## Dependencies
05, 06. (Consumes `NetworkPolicy` from 05 for `unshare_all(network)` and the run types from
06.)

## Non-goals
No changeset scanning (21); no Landlock/seccomp; no macOS.

## Design References
DESIGN §3.5 (overlay row), §6.1 (overlay detection), §13 (run dir), §10.2 T2/T5/T9,
research/01 (userns/overlayfs/userxattr), ADR-001/003/004.
