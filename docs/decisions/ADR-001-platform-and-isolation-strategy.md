# ADR-001: Platform support and sandbox isolation strategy

- Status: Accepted
- Date: 2026-07-10
- Deciders: user (product owner), Fable (design)
- Related: [research/01-sandbox-primitives.md](../research/01-sandbox-primitives.md), ADR-003, ADR-004

## Context

dryrun rehearses a shell command in a sandbox and shows the filesystem changes as a diff
before the real run. Two coupled decisions drive the whole architecture: which OSes we
support, and how we isolate the rehearsal. The user selected **macOS + Linux both** and
**OS-native, no external runtime dependency** during requirements clarification.

Constraints established:

- No container runtime (Docker/Podman/OrbStack) may be required. Rehearsal fidelity to
  "what happens on *my* machine" is a core value; running inside a foreign OS image
  breaks it.
- Must run unprivileged (no sudo) in the common case.
- Ubuntu 24.04 restricts unprivileged user namespaces by default (see research doc).

## Decision

Support macOS and Linux behind a single `SandboxEngine` trait, with three concrete
engines selected at runtime by a capability probe:

1. **`macos-seatbelt`** (macOS 13+, APFS): APFS `clonefile` shadow of the workspace +
   Seatbelt SBPL profile applied via `sandbox_init` in the child.
2. **`linux-overlay`** (kernel >= 5.13, user namespaces permitted): user+mount+pid+net
   namespaces + overlayfs mounted at the workspace path + Landlock write confinement +
   loopback-only netns.
3. **`linux-portable`** (any kernel >= 5.13, no userns needed): shadow copy + Landlock
   write confinement + seccomp socket-family deny. This is the fallback when user
   namespaces are blocked (e.g. Ubuntu 24.04 default, locked-down CI).

A non-APFS or pre-13 macOS degrades to a copy-based shadow while keeping the Seatbelt
profile; if Seatbelt cannot initialize, the run is refused (fail-closed), never silently
run unsandboxed.

Engine selection precedence, failure semantics, and the capability probe are specified in
DESIGN.md §Engine selection.

## Consequences

Positive:

- Zero runtime dependencies; single static-ish binary per platform.
- High fidelity: rehearsal touches a copy-on-write view of the user's actual files.
- The trait boundary isolates unsafe OS code and makes future engines (e.g. FreeBSD, a
  Seatbelt replacement) additive.

Negative / accepted costs:

- Three engines to build, test, and security-review instead of one.
- Guarantees are **not identical across engines** (e.g. overlay captures deletes via
  whiteouts natively; the copy engines reconstruct deletes by tree diff). DESIGN.md has a
  per-engine guarantee matrix and every user-facing report names the engine used.
- The rehearsal executes at a shadow path, so a command writing to a hard-coded absolute
  path inside the workspace behaves differently on copy engines vs overlay. Documented as
  a platform caveat and surfaced in reports.
- Dependence on the deprecated-but-functional Seatbelt API is a tracked known unknown.

## Alternatives considered

See [research/01-sandbox-primitives.md](../research/01-sandbox-primitives.md) "Options
considered and rejected" (containers, ptrace, LD_PRELOAD, Endpoint Security, macFUSE,
whole-system shadow).
