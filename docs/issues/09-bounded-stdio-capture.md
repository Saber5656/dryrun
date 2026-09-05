# 09 — Bounded stdio capture

## Title
Bounded stdio capture

## Summary
Implement `StdioSinks`: tee the rehearsed child's stdout/stderr to the user's terminal
while capturing up to a per-stream byte cap, setting a truncation flag when the cap is hit
(DESIGN §4.2 `--max-output`, §11 output-cap row).

## Context
Captured output feeds the report and is shown live. A hostile command can emit unbounded
output (DESIGN §10.2 T5); capture must be bounded and never OOM the tool, while the child
keeps running to its other limits.

## Scope
- In: `src/engine/common/stdio.rs` — `StdioSinks`, the tee logic, truncation flags.
- Out: process spawning (engines), wall-clock/disk limits (10).

## Detailed Requirements
1. `StdioSinks` owns two bounded buffers (stdout, stderr) plus optional passthrough to the
   real terminal (unless `--json`/`--quiet` suppress live echo — decided by policy passed
   in).
2. Reading model: engines give the child pipe read-ends; a small thread per stream (or
   `poll`) reads chunks, (a) writes them to the live terminal if enabled, (b) appends to
   the bounded buffer until `max_output_bytes`, after which it sets `truncated=true` and
   stops buffering but **keeps draining** the pipe (so the child does not block on a full
   pipe).
3. Never allocate more than `max_output_bytes` (+ small constant) per stream. Use a
   `Vec<u8>` with a hard cap; drop excess.
4. Expose `into_captured() -> Captured { stdout: Vec<u8>, stderr: Vec<u8>,
   stdout_truncated: bool, stderr_truncated: bool }`.
5. Handle non-UTF-8 output (store bytes; renderers lossily convert for display).
6. Clean shutdown: joins reader threads on child exit; no leaked fds/threads (tested).

## Acceptance Criteria
- A child emitting > cap bytes yields `truncated=true`, buffer length ≤ cap, and the child
  is not deadlocked (it runs to completion).
- Live echo appears when enabled and is suppressed under `--json`/`--quiet`.
- Non-UTF-8 bytes are captured without loss and without panic.
- No thread/fd leaks (a stress test spawning many captures stays flat).

## Validation
- `cargo test engine::common::stdio` with a helper child (`printf`/`yes | head` style or a
  Rust test binary) that emits > cap; assert cap + truncation + completion.

## Dependencies
06.

## Non-goals
No timeout/disk logic (10); no sandboxing.

## Design References
DESIGN §3.1 (tee), §4.2 (`--max-output`), §10.2 T5, §11 (output cap).
