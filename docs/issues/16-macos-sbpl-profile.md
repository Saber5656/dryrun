# 16 — macOS SBPL profile builder

## Title
macOS SBPL (Seatbelt) profile builder

## Summary
Generate the Seatbelt SBPL profile string that allows broad reads, confines writes to the
shadow + private tmp, and denies network by default (allow-default with targeted denies, to
survive Tahoe sysctl regressions — research/01, ADR-003/004).

## Context
The profile is the macOS enforcement of workspace confinement and network default-deny. An
allow-default-with-targeted-denies shape is chosen deliberately (research/01: deny-default
profiles break when shells read new sysctls on Tahoe).

## Scope
- In: `src/engine/macos/seatbelt.rs` — `build_profile(layout, policy) -> String` producing
  SBPL text; a golden snapshot of a representative profile.
- Out: applying it (15/17); shadow creation (11).

## Detailed Requirements
Inputs: `build_profile` takes plain values — the writable/shadow subpaths (`&[PathBuf]`),
the private tmp path, and the `NetworkPolicy` — NOT the `ShadowLayout` type, so this issue
does not depend on 11's code (only on 05's `NetworkPolicy`). The caller (17) passes the
concrete paths from the layout.

1. Emit SBPL (`(version 1)`) with:
   - a permissive base (allow default) so ordinary reads/execs/sysctls work (avoids the
     Tahoe deny-default breakage),
   - `(deny file-write*)` globally, then `(allow file-write* (subpath "<path>"))` for each
     writable/shadow subpath and for `<private_tmp>`,
   - **Network (matches ADR-003 / user decision 2026-07-11 exactly)**: when
     `policy.network == Deny`, deny external IP while allowing local IPC + loopback:
     `(deny network-outbound)` and `(deny network-inbound)` as the base, then
     `(allow network-outbound (remote unix))` / `(allow network-bind (local unix))` for unix
     sockets and `(allow network-outbound (remote ip "localhost:*"))` +
     `(allow network-inbound (local ip "localhost:*"))` for loopback. Net effect: unix-socket
     local IPC and loopback work; non-local IP is denied. When `policy.network == Allow`,
     omit the network denies entirely (full outbound). Do NOT leave the profile in an
     inconsistent "prefer no network" state — the three engines must present the same
     policy (external denied, local IPC + loopback allowed).
   - the private tmp is the only writable temp: `$TMPDIR` is pointed at `<private_tmp>` by
     the engine (17). A hard-coded write to real `/tmp/...` is NOT in the allow list, so it
     is denied by `(deny file-write*)` and surfaces as an out-of-workspace blocked write
     (ADR-004 /tmp decision) — do not add a blanket `(allow file-write* (subpath "/tmp"))`.
2. Properly escape paths in SBPL (quotes, special chars); reject paths that cannot be
   safely represented (fail-closed).
3. Provide report hooks: writes attempted outside the allowed subpaths are denied; where
   Seatbelt can be put in a reporting/trace mode to enumerate denials for the
   `blocked_writes` list, do so; otherwise document that macOS blocked-write enumeration is
   best-effort and add the caveat (DESIGN §6.1 note).
4. Keep the profile deterministic for a given layout (golden-testable).
5. Do NOT hardcode user-specific absolute paths in the committed golden; use placeholders
   normalized in the snapshot.

## Acceptance Criteria
- Generated profile denies writes outside the shadow/tmp and allows writes inside (verified
  behaviorally in 17's integration test).
- Hard-coded `/tmp/...` write is denied by the profile (not allow-listed); a `$TMPDIR` write
  (pointed at private tmp) is allowed.
- External IP network denied by default; local IPC (unix) and loopback allowed by default;
  all network allowed under `--allow-network` (consistent with ADR-003).
- Paths are correctly escaped; an un-representable path fails closed (error, not a silent
  broad allow).
- Golden snapshot of a normalized profile is stable.

## Validation
- `cargo test engine::macos::seatbelt` (unit: string shape + escaping + network toggle +
  golden). Behavioral enforcement is validated in 17/31 on macOS.

## Dependencies
05.

## Non-goals
No FFI/apply (15); no process spawn (17); no Linux.

## Design References
DESIGN §6.1 (blocked-write caveat), §10.2 T1/T2, §10.7, research/01 (Tahoe deny-default
risk), ADR-003/004.
