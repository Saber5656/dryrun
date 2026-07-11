# 06 — SandboxEngine trait and core run types

## Title
SandboxEngine trait and core run types

## Summary
Define the `SandboxEngine` trait and the shared types it uses (`EngineKind`, `Probe`,
`RunContext`, `ExecOutcome`, `Command`, `StdioSinks`) from DESIGN §3.3. This is the
contract every engine implements and the orchestrator drives.

## Context
The trait isolates all unsafe OS code behind one stable interface (DESIGN §3.3, ADR-001).
Locking it down early lets the three engines and the orchestrator be built in parallel.

## Scope
- In: `src/engine/mod.rs` with the trait + types + a `NoopEngine` test double.
- Out: real engines (17/19/21), selection (07), monitor/stdio impls (09/10).

## Detailed Requirements
1. `EngineKind { MacosSeatbelt, LinuxOverlay, LinuxPortable }` with `Display`/`as_str`
   producing the exact strings in DESIGN §7 (`"macos-seatbelt"`, etc.) and `FromStr` for
   `--engine`.
2. `Probe { usable: bool, reasons: Vec<String>, degraded: Vec<String> }` (DESIGN §3.3).
3. `RunContext`: holds paths created at `prepare` — `run_dir`, `work_dir` (where the child
   `chdir`s: overlay merged mount, or shadow root), `private_tmp`, `manifest_path`
   (copy engines), `engine: EngineKind`, and an opaque per-engine handle
   (`Box<dyn Any + Send>` or an enum) for teardown state (mounts, fds).
4. `Command`: `spec: CommandSpec`, resolved `program`/`args` (argv) or shell string, plus
   `env` policy (v1: inherit parent env; document that env is passed through — a v2 idea
   is env scrubbing). Provide a `to_exec()` returning program + argv (wrapping shell form
   as `["sh","-c",string]`).
5. `StdioSinks`: bounded writers for stdout/stderr provided by 09; the trait's `execute`
   writes child output into them. Define the interface here; impl in 09.
6. `ExecOutcome`: `exit: ChildExit` (`Exited(i32)` | `Signaled(i32)`), `limit_hit:
   Option<LimitKind>`, `stdout_truncated: bool`, `stderr_truncated: bool`, `duration:
   Duration`.
7. The `SandboxEngine` trait exactly as DESIGN §3.3 (`kind`, `probe`, `prepare`,
   `execute`, `scan_changes`, `cleanup`). Document the fail-closed and idempotent-cleanup
   contracts in rustdoc on the trait.
8. `NoopEngine` (behind `#[cfg(test)]` or a `testing` feature): a fake that "creates" a
   temp run dir, "runs" nothing, and returns an empty `ChangeSet`, so the orchestrator and
   report layers can be tested without OS sandboxing.
9. A `cleanup` RAII guard helper (`RunGuard`) that calls `cleanup` on drop unless
   `disarm()`ed — used by the orchestrator (26) to guarantee teardown (DESIGN §8
   invariants).

## Acceptance Criteria
- Trait + all types compile and are exercised by `NoopEngine` in a unit test that runs the
  `prepare → execute → scan_changes → cleanup` sequence.
- `EngineKind` round-trips to/from the exact DESIGN §7 strings (test).
- `RunGuard` calls `cleanup` on drop; a test asserts cleanup runs exactly once and again
  is a no-op after `disarm`.

## Validation
- `cargo test engine::mod` including the `NoopEngine` lifecycle and `RunGuard` drop test.

## Dependencies
01, 02.

## Non-goals
No OS-specific code; no selection logic; no real capture.

## Design References
DESIGN §3.1 (process model), §3.3 (trait), §8 (cleanup invariant), §7 (kind strings),
ADR-001.
