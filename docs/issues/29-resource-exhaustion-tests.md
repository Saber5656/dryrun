# 29 — Resource-exhaustion tests

## Title
Resource-exhaustion tests

## Summary
Prove the monitor (10) and engines contain hostile resource use: fork bombs are reaped,
disk/inode budgets abort the run, wall-clock timeout fires, output caps hold, and no child
survives teardown (DESIGN §10.2 T5/T9, §11).

## Context
A rehearsed command is adversarial; the tool must never be taken down or leave a mess by a
runaway command. These tests lock in the resource controls.

## Scope
- In: `tests/limits_*.rs` (integration), engine/os-gated.
- Out: the monitor impl (10); escape tests (28).

## Detailed Requirements
1. Scenarios (named tests), each asserting the run terminates promptly and reports the right
   `LimitKind`/exit 43 (or reaps cleanly):
   - infinite loop (`while true; do :; done`) with `--timeout 1s` → killed, `Timeout`,
     no survivor.
   - disk filler (write zeros) with small `--max-disk` → killed, `Disk`.
   - inode filler (create many files) with small `--max-files` → killed/`Inodes` or
     truncated changeset with the flag, per DESIGN §11 (assert the documented behavior).
   - output flood (`yes`) with small `--max-output` → captured buffer capped,
     `truncated=true`, run still terminates.
   - fork bomb (bounded in test: spawn N children that sleep) → whole process group killed
     at teardown; assert zero survivors (check the pgid is gone).
   - background daemon (`sleep 300 &`) → killed at cleanup; no survivor.
2. Survivor check: after `cleanup`, assert no process in the run's process group exists and
   no run-dir residue remains.
3. Keep tests bounded/safe for CI (small limits, short timeouts, capped fork counts) so they
   cannot actually exhaust the CI host.

## Acceptance Criteria
- Every scenario terminates within a few seconds and reports the documented outcome.
- No orphaned processes or run-dir residue after any scenario.
- On the overlay engine, the pid namespace ensures the fork bomb/daemon are fully reaped.

## Validation
- `cargo test --test limits_timeout --test limits_disk --test limits_output ...` (os-gated),
  wired into CI (33).

## Dependencies
10, 26, and at least one engine.

## Non-goals
No escape (28); no redaction (30); no network (31).

## Design References
DESIGN §10.2 T5/T9, §11 (failure modes), §4.2 (limits), §15.
