# 26 — Orchestrator and run state machine

## Title
Orchestrator and run state machine

## Summary
Wire the full pipeline from DESIGN §3.1/§8 in `cli/run.rs`: parse → config → policy →
select engine → prepare → execute → scan → render → decide → (drift → real) → cleanup, with
the cleanup RAII guarantee, signal handling, and correct exit-code mapping for every path.

## Context
This is the integration point that turns all prior modules into the working `dryrun`
command and enforces the state-machine invariants (DESIGN §8), especially that cleanup
always runs and no real execution happens before approval.

## Scope
- In: `src/cli/run.rs` — the `run(args) -> ExitCode` orchestrator + signal handling +
  minimal logging setup.
- Out: the modules it composes (03/04/05/07 + engines + 09/10/22/23/24/25/27).

## Detailed Requirements
1. Implement the state machine in DESIGN §8 exactly, including transitions to `FAILED(code)`
   and the `POLICY_VIOLATION` (30) emission from `SCANNED` when `violation_mode == Fail`
   and blocked writes / network attempts exist.
2. Cleanup guarantee: wrap the engine `RunContext` in the `RunGuard` (06) so `cleanup` runs
   on every exit path — normal, error, signal, panic. Decide the panic strategy here: if
   `panic = "abort"` is desired for release (36), ensure teardown still happens (e.g. a
   signal-safe best-effort cleanup or keep unwinding for the sandbox path); document and
   test that a panic mid-rehearsal does not leave mounts/processes/run-dirs behind.
3. Signal handling: SIGINT/SIGTERM during rehearsal → kill child process group (10),
   cleanup, exit with a mapped code. Do not leave orphaned children or mounts.
4. Logging: initialize a minimal stderr logger honoring `-v/-vv/-q`; keep stdout pure under
   `--json`.
5. Route subcommands: `doctor`→27, `version`→01, `completions`→34.
6. Exit-code mapping: every terminal state maps through 02 to the correct code; the JSON
   report (22) mirrors the code. `OK` (0) for successful rehearsal+approved real run (or
   `--no-execute`, or NoChannel).
7. Ensure ordering invariant: `run_real` (25) is unreachable before an Approve `Decision`
   (24) — encode with types (pass an `Approved` token) or an explicit assert + test.

## Acceptance Criteria
- End-to-end on Linux (portable engine): `dryrun -c 'touch x'` in a temp dir shows a
  `Created` change, prompts (or respects `--yes`/`--no-execute`), and on approval actually
  creates `x`; exit codes match DESIGN §4.3 for each branch.
- Cleanup runs on success, on error, on SIGINT, and on an injected panic (tested: no
  leftover run dir / mount / child process).
- `violation_mode == Fail` with a blocked write → exit 30.
- `--json` yields a pure JSON object and the mirrored exit code.
- Real run never occurs without approval (test attempts every non-approve path).

## Validation
- `cargo test --test e2e` (os-gated to the available engine) covering the branches above;
  a panic-injection test asserts teardown; a SIGINT test asserts no orphans.

## Dependencies
02, 03, 04, 05, 06, 07, 08, 22, 23, 24, 25, and at least one engine (17 on macOS; 19 or 21
on Linux). (02 for exit mapping, 06 for `RunGuard`/run types.)

## Non-goals
No new product features; no doctor internals (27).

## Design References
DESIGN §3.1 (pipeline), §8 (state machine + invariants), §4.3 (exit codes), §11 (failure
modes), §14 (logging).
