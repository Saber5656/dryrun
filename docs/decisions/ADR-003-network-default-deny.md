# ADR-003: Network access is denied by default during rehearsal

- Status: Accepted
- Date: 2026-07-10
- Deciders: user (product owner), Fable (design)
- Related: ADR-001, ADR-004, DESIGN.md §Security model

## Context

The premise of dryrun is that a rehearsal can be *discarded*. Filesystem writes are
captured in a shadow and thrown away, so they are safe to allow. **Network side effects
are different: they cannot be rolled back.** A rehearsed command that sends an HTTP POST,
publishes a package, sends an email, or deletes a remote object has already caused the
irreversible effect the tool exists to prevent. The user selected **network blocked by
default** during clarification.

## Decision

During rehearsal, deny **external** network by default while allowing local IPC. The one
network policy, applied consistently across engines (user decision 2026-07-11, "pragmatic:
allow local IPC"):

- **Denied by default**: outbound IP networking — `AF_INET`, `AF_INET6`, `AF_PACKET`.
- **Allowed by default**: local IPC (`AF_UNIX`, `AF_NETLINK`) and loopback where the engine
  provides it for free, so ordinary tools that use unix sockets / bind localhost keep
  working.

Per engine:

- `linux-overlay`: a network namespace with only a loopback interface up (no route out).
  Loopback and local IPC work; external IP is unreachable by construction.
- `linux-portable`: seccomp filter denying `socket(2)`/`socketcall` for
  `AF_INET`/`AF_INET6`/`AF_PACKET`; `AF_UNIX`/`AF_NETLINK` allowed. Because there is no
  network namespace, a host-local daemon reachable over a unix socket (e.g. a running
  proxy, `docker.sock`) remains reachable — a documented residual exfiltration caveat of
  the portable engine that the overlay engine does not have. Note: with `AF_INET` blocked
  at `socket()` time, loopback **TCP** is unavailable on this engine (a caveat vs overlay);
  unix-socket localhost IPC still works.
- `macos-seatbelt`: SBPL denies `network-outbound`/`network-inbound` to non-local
  addresses while allowing `network*` to `localhost`/unix sockets (local IPC + loopback
  allowed, external denied).

An explicit `--allow-network` flag re-enables full outbound network for the rehearsal. When
a command fails *because* external network was blocked, dryrun detects the signature (e.g.
connection refused / DNS failure right before a non-zero exit for known package managers)
and tells the user they can re-run with `--allow-network`.

The **real** run performed after approval is a normal process with normal network access
(ADR-005); default-deny applies only to the sandboxed rehearsal.

## Consequences

Positive:

- The tool cannot cause an unrecoverable remote side effect during the "safe preview"
  step. This is the property that makes dryrun trustworthy.
- Default-deny is the secure default; enabling network is a conscious, logged choice.

Negative / accepted costs:

- Commands that must reach the network to do anything meaningful (`npm install`, `git
  fetch`, `curl`) produce an incomplete rehearsal by default. Mitigated by clear
  detection + the `--allow-network` hint, documented prominently.
- With `--allow-network`, rehearsal can cause real remote effects; this is the user's
  explicit opt-in and is surfaced in the plan header and the report.
- **Portable-engine residual local-IPC exfiltration**: allowing `AF_UNIX`/`AF_NETLINK`
  means a hostile command on the portable engine could reach a host-local daemon that
  itself has network access. Accepted for tool compatibility; documented in the report
  caveats, `SECURITY.md`, and the engine guarantee matrix. Users who need the stronger
  guarantee should prefer the overlay engine (netns isolates external egress fully) or a
  future strict mode (v2).

## Alternatives considered

- **Allow network by default with a warning**: rejected. A warning does not undo a POST;
  it makes the tool unsafe by default, defeating its purpose.
- **Interactive prompt on first blocked socket**: deferred. Creates a dual
  interactive/non-interactive code path and complicates the agent/JSON mode; the flag +
  detection hint covers the need for v1.
