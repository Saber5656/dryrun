# 19 — linux-portable engine

## Title
linux-portable engine

## Summary
Implement the `linux-portable` `SandboxEngine`: shadow copy (11) + Landlock write
confinement (13) + seccomp network deny (18), no user namespaces required — the fallback
that works under Ubuntu 24.04's userns restriction (research/01, ADR-001).

## Context
This engine is the guaranteed-available Linux path and the earliest end-to-end vertical
slice (ISSUE_PLAN waves). It must run where the overlay engine cannot.

## Scope
- In: `src/engine/linux/portable.rs` — the `SandboxEngine` impl.
- Out: its building blocks (11/12/13/18); overlay engine (21).

## Detailed Requirements
1. `probe`: Linux, kernel ≥ 5.13, Landlock present, seccomp available. No userns needed.
   Reflink-capability note → degraded copy caveat.
2. `prepare(policy)`: run dir (0700, DESIGN §13 Linux location under `$XDG_STATE_HOME`),
   shadow(s) + private tmp + manifest (11). `work_dir = shadow root`.
3. `execute(ctx, cmd, io)`:
   - `fork`; child: new process group; stdio pipes; **close all inherited file descriptors
     except stdio (0/1/2)** before applying filters (so no pre-opened external socket leaks
     into the sandbox — closes the local-IPC-residual hole from 18); `chdir(shadow)`; point
     `TMPDIR` at the private tmp (real `/tmp/...` hard-coded writes are NOT allow-listed in
     Landlock and are therefore denied+reported per ADR-004); set `no_new_privs`; apply
     seccomp (18) then Landlock (13) (order: seccomp then Landlock, both before exec); on any
     failure fail-closed (42 sentinel); `execvp`/`sh -c`.
   - parent: monitor (10) on pgid, tee stdio (09), `waitpid`, build `ExecOutcome`.
4. `scan_changes`: tree-diff scanner (12) vs manifest; `blocked_writes` from Landlock
   EACCES observations where feasible (else caveat); `network_attempts` from seccomp tap
   where feasible (else caveat); add caveats (`shadow path`, `renames best-effort`).
5. `cleanup`: kill surviving pgid, remove run dir; idempotent; confined to run dir.
6. Minimal `unsafe`, `// SAFETY:` on fork/exec/dup2.

## Acceptance Criteria
Environment: Linux.
- `touch newfile` → `Created`, real workspace unchanged.
- `rm existing` → `Deleted`, real file survives.
- Write to `$HOME/x` denied (real `$HOME` unchanged); appears in `blocked_writes` or the
  caveat is present.
- Hard-coded write to real `/tmp/x` is denied (real `/tmp/x` not created); a `$TMPDIR` write
  succeeds into the private tmp. (ADR-004)
- External IP network denied by default (AF_INET socket fails); AF_UNIX works; allowed with
  `--allow-network`. Inherited non-stdio fds are closed before exec (assert a pre-opened
  socket fd is not usable in the child).
- Runs on a userns-restricted host (AppArmor) — this is the key differentiator; a CI job
  (33) exercises it with unprivileged userns disabled.
- Sandbox-apply failure → 42, never unconfined exec.

## Validation
- `cargo test --test linux_portable_it` (os-gated) + the shared conformance table; CI job
  with `kernel.apparmor_restrict_unprivileged_userns=1` (or equivalent) to prove it works
  when overlay cannot.

## Dependencies
05, 06, 09, 10, 11, 12, 13, 18.

## Non-goals
No overlayfs/namespaces (20/21); no macOS; no rendering.

## Design References
DESIGN §3.5 (portable row), §6.1, §13, §10.2 T1/T2/T3, research/01, ADR-001/003/004.
