# 07 — Engine selection and capability probe

## Title
Engine selection and capability probe

## Summary
Implement `engine::select::choose` and the per-engine capability probes with the
fail-closed precedence in DESIGN §3.4, plus the `SelectionReport` shown in reports and by
`doctor`.

## Context
Selection is a security control (DESIGN §10.2 T3/T10): a forced-but-unusable engine must
error, and no fallback may silently run unsandboxed. On Linux, overlay→portable fallback
must be automatic when user namespaces are blocked (Ubuntu 24.04 default,
research/01).

## Scope
- In: `src/engine/select.rs` — probes (non-mutating), `choose`, `SelectionReport`.
- Out: the engines' real `prepare/execute` (17/19/21); `doctor` rendering (27).

## Detailed Requirements
1. Each engine exposes a cheap `probe(&self) -> Probe` (from 06). Probes MUST NOT mutate
   system state. Concretely:
   - macOS: OS is macOS; detect APFS on the workspace volume (degraded=copy if not);
     confirm `sandbox_init` symbol available. `usable` unless Seatbelt unavailable.
   - linux-overlay: kernel ≥ 5.13; can create a user namespace (attempt an `unshare`
     probe in a child that immediately exits — must not leave state); Landlock present;
     overlayfs available. If userns creation returns EPERM (AppArmor), `usable=false`,
     `reasons` includes the Ubuntu remediation hint.
   - linux-portable: kernel ≥ 5.13; Landlock present; seccomp available. `usable` almost
     always on Linux; `degraded=copy fallback` note if workspace fs lacks reflink.
2. `choose(policy) -> Result<(Box<dyn SandboxEngine>, SelectionReport), DryrunError>`
   implementing DESIGN §3.4 precedence exactly:
   - `--engine` forced + unusable → `DryrunError::Config` (40).
   - macOS → seatbelt; if not usable → `SandboxInit` (42) (never unsandboxed).
   - Linux → overlay if usable else portable; if neither → `UnsupportedPlatform` (41) with
     remediation text.
   - other OS → `UnsupportedPlatform` (41).
3. `SelectionReport { chosen: EngineKind, reason: String, degraded: Vec<String>, rejected:
   Vec<(EngineKind, String)> }` — serialized into DESIGN §7 `meta.engine_selection`.
4. Remediation strings for the Ubuntu-userns-blocked case must name the concrete options
   from research/01 (install shipped AppArmor profile, or the admin sysctl opt-out) and
   point to `dryrun doctor`.

## Acceptance Criteria
- On Linux with userns available, `choose` picks overlay; with userns blocked (simulated
  by a probe injection point), it picks portable and records overlay in `rejected` with a
  reason.
- Forcing an engine whose probe is unusable → exit 40; forcing a usable one uses it.
- On macOS non-APFS, seatbelt is chosen with a `degraded` copy-fallback note.
- `SelectionReport` serializes to the exact DESIGN §7 shape.

## Validation
- `cargo test engine::select` with probe injection (trait object / function pointer) to
  simulate userns-blocked and non-APFS without needing those environments in unit tests.
- Integration coverage of the real probes lands in 33 (CI matrix).

## Dependencies
05, 06. (`choose(policy)` consumes `EffectivePolicy` from 05 and the trait/types from 06.)

## Non-goals
No prepare/execute; no doctor UI (27 renders this report).

## Design References
DESIGN §3.4 (selection), §3.5 (matrix), §7 (engine_selection), §10.2 T3/T10, §11,
research/01, ADR-001.
