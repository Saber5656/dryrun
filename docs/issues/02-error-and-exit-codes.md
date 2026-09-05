# 02 — Error types and exit-code mapping

## Title
Error types and exit-code mapping

## Summary
Define the crate's typed error enum and the single, tested mapping from errors/outcomes to
the stable process exit codes in DESIGN §4.3. Exit codes are a public contract; this issue
locks them down.

## Context
Agents and scripts depend on dryrun's exit codes (DESIGN §1.3, §4.3). We need one
authoritative `DryrunError` and one function that maps every terminal condition to a code,
with tests, so later issues never invent ad-hoc codes.

## Scope
- In: `src/error.rs` with `DryrunError` (thiserror), an `ExitCode`-producing mapping, and
  a `Phase` enum (rehearsal/real) used in reports.
- Out: emitting these errors from real logic (later issues consume this module).

## Detailed Requirements
1. `DryrunError` (derive `thiserror::Error`, `Debug`) variants covering every non-zero
   code in DESIGN §4.3:
   - `RehearsalCommandFailed { child_exit: i32 }` → 10
   - `DeniedByUser` → 20
   - `PolicyViolation { blocked_writes: usize, network_attempts: usize }` → 30
   - `Config { msg: String }` → 40
   - `UnsupportedPlatform { msg: String }` → 41
   - `SandboxInit { msg: String }` → 42
   - `LimitExceeded { which: LimitKind }` → 43
   - `RealRunFailed { child_exit: i32 }` → 50
   - `Internal { msg: String }` → 70 (also the `From` target for unexpected `anyhow`/io)
   - `LimitKind` enum: `Timeout | Disk | Inodes | Output`.
2. `pub fn exit_code(err: &DryrunError) -> u8` returning the numeric code, plus
   `pub fn exit_name(code: u8) -> &'static str` returning the DESIGN §4.3 name.
3. Success codes: `OK = 0` handled by the orchestrator, not this enum. Provide a
   `pub const OK: u8 = 0;` for symmetry.
4. `impl From<std::io::Error> for DryrunError` → `Internal` unless the caller wraps it more
   specifically.
5. Doc-comment each variant with the exact DESIGN §4.3 row it implements.
6. Provide `pub fn as_report_exit(&self, phase: Phase) -> ReportExit` producing the
   struct that issue 22 serializes (`{phase, code, name, child_exit?, signal?, limit_hit?}`).
   Define `ReportExit` here or in a shared types module; keep field names identical to
   DESIGN §7.

## Acceptance Criteria
- Every non-zero code in DESIGN §4.3 has exactly one producing variant; a unit test
  asserts the full code↔name table matches DESIGN §4.3 (no missing/extra codes).
- `exit_code`/`exit_name` are total (compile-time exhaustive over variants).
- Mapping is covered by a table-driven unit test.

## Validation
- `cargo test error::` passes.
- A test iterates all codes {10,20,30,40,41,42,43,50,70} and asserts `exit_name` matches
  DESIGN §4.3.

## Dependencies
01.

## Non-goals
No error is actually raised by product logic here; no logging setup.

## Design References
DESIGN §4.3 (exit codes), §7 (report exit block), §11 (failure modes).
