# 21 — linux-overlay engine and whiteout scan

## Title
linux-overlay engine and whiteout scan

## Summary
Implement the `linux-overlay` `SandboxEngine`: set up namespaces + overlayfs (20), apply
Landlock (13), run the command at the overlay-merged workspace, and scan the overlay
upperdir (including whiteouts/opaque dirs) into a `ChangeSet` — the highest-fidelity engine
(DESIGN §3.5, §6.1).

## Context
Overlay captures writes natively in the upperdir and encodes deletes as whiteouts, giving
better fidelity than copy engines (research/01). This engine is preferred on capable Linux.

## Scope
- In: `src/engine/linux/overlay.rs` — the `SandboxEngine` impl + upperdir scanner.
- Out: namespace/overlay primitives (20), Landlock (13), monitor/stdio (09/10).

## Detailed Requirements
1. `probe`: from 07 (userns creatable, overlayfs + Landlock present, kernel ≥ 5.13).
2. `prepare(policy)`: run dir (0700); create overlay `upper`/`work`; do NOT copy the
   workspace (overlay lowerdir is the real workspace, read-only lower). Reserve private tmp.
3. `execute(ctx, cmd, io)` — use the `spawn_pid1` sequence from 20 (do NOT assume the
   unsharing process becomes pid 1):
   - parent process: `unshare_all(network)` sets up user+mount+net namespaces and requests
     the pid namespace, then calls `spawn_pid1(closure)` which `fork`s the PID-1 child.
   - PID-1 child (namespace init): `mount_proc`, `mount_overlay` (bind RO lower → overlay
     over workspace), `mount_private_tmp`, `bring_up_loopback` (20); then `fork/exec` the
     command in its own process group after closing inherited non-stdio fds, `chdir` to the
     merged workspace, `no_new_privs`,
     Landlock (13) with write-roots = merged workspace + private tmp; reap the command and
     any orphans; relay the command's exit status to the parent via the pipe. Fail-closed on
     any setup error (42 sentinel written to the pipe).
   - parent: start the monitor (10) targeting the PID-1 child's process group, tee stdio
     (09), wait for the PID-1 child, and read the relayed command status → `ExecOutcome`.
4. `scan_changes`: walk the upperdir:
   - regular file in upper → compare to lowerdir counterpart by hash: absent below =
     `Created`, present+differ = `Modified` (feed 14 for diff), present+same-content but
     mode differs = `ModeChanged`.
   - char device `0:0` = whiteout → `Deleted`.
   - dir with `user.overlay.opaque` (or `trusted.overlay.opaque` per mount type) xattr =
     directory replaced → represent children accordingly.
   - rename detection best-effort (hash pairing) with caveat.
   - `blocked_writes` from Landlock EACCES observations (best-effort; caveat if
     incomplete); `network_attempts` best-effort (netns makes external connects fail;
     caveat notes enumeration limits). Do NOT add the `shadow path` caveat when the overlay
     is mounted at the real path; DO add it if the fallback merged-dir path was used (20).
5. `cleanup`: unmount overlay/tmp/proc in order (20 helpers), kill surviving pgid, remove
   run dir; idempotent; confined to run dir; MUST NOT touch the lowerdir (real workspace).
6. Verify whiteout/opaque encoding on the CI kernel (DESIGN §2.4 U4); if a kernel encodes
   differently, handle both `user.` and `trusted.` xattr namespaces.

## Acceptance Criteria
Environment: Linux, user namespaces enabled.
- `touch newfile` → `Created` from upperdir; real workspace unchanged (lowerdir intact).
- `rm existing` → `Deleted` via whiteout; real file survives.
- `chmod` only → `ModeChanged`.
- Directory replace produces coherent changes (opaque handled).
- Out-of-workspace write denied (Landlock) and network denied (netns) by default.
- `cleanup` unmounts everything and leaves the real workspace byte-identical (asserted by
  hashing the workspace before/after).

## Validation
- `cargo test --test linux_overlay_it` (os+userns gated) + shared conformance table; assert
  lowerdir/workspace unchanged post-run. CI userns-enabled job runs it.

## Dependencies
05, 06, 08, 09, 10, 13, 20.

## Non-goals
No copy-engine scanning (12); no macOS; no rendering.

## Design References
DESIGN §3.5 (overlay row + caveat), §6.1 (overlay detection: whiteout/opaque), §2.4 U4,
§10.2 T1/T2/T3/T9, research/01, ADR-001/003/004.
