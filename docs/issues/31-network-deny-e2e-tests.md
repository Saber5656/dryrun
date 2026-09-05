# 31 — Network default-deny end-to-end tests

## Title
Network default-deny end-to-end tests

## Summary
Prove that during rehearsal, outbound network is blocked by default on every engine and
permitted only with `--allow-network`, and that the "blocked by network" hint is surfaced
(DESIGN §10.2 T2, ADR-003).

## Context
Network default-deny is the property that makes a rehearsal safe to discard (ADR-003).
These tests verify it per engine and confirm the opt-in path works.

## Scope
- In: `tests/network_*.rs` (engine/os-gated), using a local listener to avoid real external
  traffic.
- Out: seccomp/netns/SBPL impls (18/20/16).

## Detailed Requirements
1. Use a loopback TCP listener started by the test as the "network" target so no real
   external endpoints are contacted. (Loopback is allowed by design; to test the deny path,
   also attempt a connect to a non-loopback address/port that the engine must block —
   assert it fails without leaving the host, e.g. connect to a TEST-NET address which
   should be unreachable/blocked.)
2. Scenarios per engine:
   - default: a helper that attempts an outbound (non-loopback) TCP connect fails (blocked);
     the rehearsal completes; if the command signals network failure, the report/hint
     mentions `--allow-network`.
   - `--allow-network`: assert the network *stack is enabled* — i.e. creating an `AF_INET`
     socket and connecting to the test's loopback listener succeeds (which fails by default
     on the portable engine where `AF_INET` is blocked). This proves the flag re-enables IP
     socket creation. (A true non-loopback outbound connection cannot be asserted
     hermetically; if real outbound must be proven, add an optional, clearly-marked
     non-hermetic CI job that connects to a known external host, off by default.)
   - overlay engine: assert only `lo` exists in the netns (no external route).
   - portable engine: assert `AF_INET` socket creation is denied by seccomp while `AF_UNIX`
     works.
   - macOS: assert SBPL network deny blocks the connect; allowed with the flag.
3. Assert the real (post-approval) run is NOT part of these tests (network deny applies to
   rehearsal only); if needed, use `--no-execute` so no real command runs.
4. Keep tests hermetic: no dependency on external network availability; a blocked attempt
   must be provable via error/behavior, not by contacting the internet.

## Acceptance Criteria
- Default rehearsal blocks outbound non-loopback network on all three engines.
- `--allow-network` permits it (loopback listener reached).
- overlay: only `lo` present; portable: INET denied / UNIX allowed; macOS: SBPL denies.
- The "re-run with --allow-network" hint appears when a command fails due to the block
  (at least for a known signature; heuristic per DESIGN §2.4 U7).

## Validation
- `cargo test --test network_default --test network_allow` (os/engine-gated), wired into CI
  (33). No external network required.

## Dependencies
26 and at least one engine (16/18/20 via 17/19/21).

## Non-goals
No real-run network testing; no escape/resource/redaction.

## Design References
DESIGN §10.2 T2, §2.4 U7 (blocked-network detection heuristic), §15, ADR-003.
