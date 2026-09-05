# 10 — Resource monitor (walltime, disk, inodes)

## Title
Resource monitor (walltime, disk, inodes)

## Summary
Implement the monitor that enforces the rehearsal's wall-clock timeout and disk/inode
budgets, killing the child's process group when a limit is hit and reporting which limit
fired (DESIGN §4.2 limits, §10.2 T5, §11).

## Context
A rehearsed command is adversarial and may loop forever, fork-bomb, or fill the disk
(DESIGN §10.2 T5). The monitor is the resource-exhaustion control shared by all engines.

## Scope
- In: `src/engine/common/monitor.rs` — timeout timer, periodic disk/inode sampling of the
  shadow/upper dir, process-group kill, `LimitKind` reporting.
- Out: pid-namespace reaping specifics (overlay engine 21), stdio caps (09).

## Detailed Requirements
1. `Monitor::start(run_dir, limits, child_pgid) -> MonitorHandle`. Runs a background
   thread that:
   - fires at `wall_timeout`: SIGTERM the child process group, then SIGKILL after a short
     grace (e.g. 2s); records `LimitKind::Timeout`.
   - samples the growing capture dir (shadow/upper/tmp) every N ms (e.g. 250ms): total
     bytes > `max_disk` → kill, `LimitKind::Disk`; total inode/file count > `max_files` →
     kill, `LimitKind::Inodes`.
2. Sampling must be cheap (incremental walk or `statfs` delta where possible) and must not
   follow symlinks out of the run dir.
3. `MonitorHandle::stop() -> Option<LimitKind>`: called after the child exits; returns the
   limit that fired, if any. Idempotent; joins the thread.
4. Process-group semantics: engines put the child in its own process group / pid namespace
   so the monitor can signal the whole tree (kill background/daemon children too — DESIGN
   §10.2 T9). This issue defines the kill helper `kill_process_group(pgid, signal)`;
   engines set up the group.
5. On any limit, cooperate with the orchestrator to set `ExecOutcome.limit_hit` and map to
   exit 43.
6. Portability: use `nix` signals; process-group kill differs per OS but the helper
   abstracts it.

## Acceptance Criteria
- A `sleep 999` rehearsal with `--timeout 1s` is killed within ~grace and reports
  `Timeout`.
- A command writing a large file with `--max-disk` small is killed and reports `Disk`.
- A command creating many files with `--max-files` small is killed and reports `Inodes`.
- A command spawning a background child has the whole group killed (no survivor after
  teardown).
- `stop()` is idempotent and leaks no thread.

## Validation
- `cargo test engine::common::monitor` (os-gated where signals differ), using small limits
  and helper children. The full fork-bomb/daemon-survivor scenarios are exercised in 29.

## Dependencies
06.

## Non-goals
No sandbox setup; no per-engine pid namespace (21 provides it, this consumes a pgid).

## Design References
DESIGN §4.2 (limits), §10.2 T5/T9, §11 (timeout/disk rows), §13 (run dir).
