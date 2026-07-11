# 18 — Linux seccomp socket-family deny filter

## Title
Linux seccomp socket-family deny filter

## Summary
Build the seccomp-BPF filter that denies `socket(2)` for `AF_INET`/`AF_INET6`/`AF_PACKET`
(allowing `AF_UNIX`/`AF_NETLINK` local IPC) for the portable engine's network default-deny,
since it cannot use a network namespace (research/01, ADR-003). Note: because `AF_INET` is
blocked at `socket()`, loopback **TCP** is unavailable on this engine (unlike the overlay
netns); only unix-socket localhost IPC works.

## Context
The portable engine runs without user namespaces, so netns is unavailable; seccomp is how
we enforce network default-deny there (DESIGN §10.2 T2).

## Scope
- In: `src/engine/linux/seccomp.rs` — `build_filter(network: NetworkPolicy) ->
  SeccompFilter` and `apply()` (child, post-fork, pre-exec).
- Out: netns (20/21), Landlock (13), macOS.

## Detailed Requirements
1. Use `seccompiler`. When `network == Deny`, build a filter that returns `EACCES` (or
   `EAFNOSUPPORT`) for `socket`/`socketcall` when the domain arg is `AF_INET`, `AF_INET6`,
   or `AF_PACKET`; allow all other syscalls (allow-default filter — we only block network
   creation, not general syscalls, to keep commands working).
   - Account for architectures where `socketcall` multiplexes (x86); on aarch64/x86_64
     `socket` is direct. Cover the target arches (x86_64, aarch64) explicitly.
2. When `network == Allow`, install no network-denying filter (still may set
   `no_new_privs`).
3. `apply` sets `no_new_privs` then installs the filter in the child before exec; on error,
   fail-closed (child exits with the 42 sentinel).
4. Optionally use `SECCOMP_RET_ERRNO` so blocked attempts return an error the command sees
   (and, where an audit tap is feasible, feed `network_attempts`); enumerating every
   attempt is best-effort — add a caveat if incomplete.
5. Provide the list of denied domains and the errno as constants; document them.
6. **Documented caveats (must be surfaced by the portable engine's report/README, not
   hidden here)**:
   - Because `AF_INET`/`AF_INET6` `socket()` calls are denied, loopback **TCP** does not
     work on this engine (unlike the overlay netns which brings up `lo`); unix-socket
     localhost IPC still works. This is the accepted per-engine loopback difference from
     ADR-003.
   - Allowing `AF_UNIX`/`AF_NETLINK` leaves a residual: a host-local daemon reachable over a
     unix socket (e.g. a proxy, `docker.sock`) is still reachable and could have its own
     network access. This is the accepted portable-engine exfil caveat (ADR-003); the
     engine (19) additionally closes inherited non-stdio file descriptors before exec so a
     pre-opened external socket cannot be inherited into the sandbox.

## Acceptance Criteria
Environment: Linux.
- With the deny filter applied, an outbound TCP connect fails (e.g. a helper that
  `socket(AF_INET,...)` returns EACCES); `AF_UNIX` sockets still work.
- With `--allow-network`, sockets are permitted.
- Filter install failure → fail-closed (42).
- Works on x86_64 and aarch64 (CI matrix covers both if available; at least x86_64).

## Validation
- `cargo test --test seccomp_it` (os-gated) with a helper that attempts INET vs UNIX
  sockets under the filter. E2E network behavior in 31.

## Dependencies
05, 06. (Consumes `NetworkPolicy` from 05 and the child-apply hook shape from 06.)

## Non-goals
No filesystem confinement (13); no namespaces (20); no macOS.

## Design References
DESIGN §10.2 T2, §10.7, research/01 (portable network deny), ADR-003.
