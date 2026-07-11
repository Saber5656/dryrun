# 17 — macos-seatbelt engine assembly

## Title
macos-seatbelt engine assembly

## Summary
Implement the `macos-seatbelt` `SandboxEngine`: prepare the shadow (11), fork a child that
applies the SBPL profile (15/16) and execs the command, capture stdio (09), enforce limits
(10), and scan changes (12).

## Context
This wires the macOS building blocks into a working engine that satisfies the DESIGN §3.5
guarantee row for macOS (ADR-001).

## Scope
- In: `src/engine/macos/mod.rs` — the `SandboxEngine` impl for macOS.
- Out: the building blocks it composes (09/10/11/12/15/16).

## Detailed Requirements
1. `probe`: macOS + `sandbox_init` available; APFS detection for degraded copy note (from
   07 shares logic or is called here). Never mutate state.
2. `prepare(policy)`:
   - create the run dir (mode 0700, DESIGN §13 macOS location under `$TMPDIR`),
   - build shadow(s) + private tmp + manifest (11),
   - build the SBPL profile (16) for the shadow/tmp/network policy,
   - return `RunContext` with `work_dir = shadow root`, tmp, manifest, and an engine handle
     holding the profile string + run dir.
3. `execute(ctx, cmd, io)`:
   - `fork`; in the child: create a new process group (for the monitor/T9), set up stdio
     pipes, close all inherited fds except stdio (0/1/2) so no pre-opened socket/file leaks
     in, `chdir(shadow)`, point `TMPDIR` at the private tmp (hard-coded real `/tmp/...`
     writes are not allow-listed in the SBPL profile and are denied+reported per ADR-004),
     apply the SBPL profile via 15 (fail-closed: on error, `_exit` with a sentinel the
     parent maps to 42), then `execvp` the command (argv form) or `sh -c` (shell form),
   - in the parent: start the monitor (10) on the child pgid, tee stdio (09), `waitpid`,
     collect `ExecOutcome` (exit/signal, limit_hit, truncation).
4. `scan_changes(ctx, policy)`: run the tree-diff scanner (12) against the manifest;
   populate `blocked_writes` from whatever Seatbelt reporting is available (else caveat);
   `network_attempts` best-effort; add the macOS caveats (`shadow path`, `renames
   best-effort`, blocked-write completeness caveat).
5. `cleanup(ctx)`: kill any surviving process group, remove the run dir; idempotent and
   confined to the run dir (never touches the real workspace).
6. All `unsafe` (fork/exec/dup2) carries `// SAFETY:` and is minimal; prefer `nix` wrappers.

## Acceptance Criteria
Environment: macOS integration.
- Rehearsing `touch newfile` in a temp workspace yields a `Created` change and does not
  create the file in the real workspace.
- Rehearsing `rm existing` yields a `Deleted` change; the real file survives.
- A write to `$HOME/x` is denied (does not modify real `$HOME`) and, where enumerable,
  appears in `blocked_writes` (else the caveat is present).
- A network attempt is blocked by default; allowed under `--allow-network`.
- Forcing sandbox-apply failure (inject a malformed profile) yields exit 42, never an
  unsandboxed exec.
- `cleanup` leaves no run-dir residue and no surviving child.

## Validation
- `cargo test --test macos_engine_it` (os-gated), covering the criteria above with temp
  workspaces. Shared conformance table also runs here (28/31).

## Dependencies
05, 06, 09, 10, 11, 12, 15, 16.

## Non-goals
No Linux; no rendering; no real (post-approval) execution (25).

## Design References
DESIGN §3.1, §3.3, §3.5 (macOS row), §6.1, §13, §10.2 T1/T2/T3/T9, ADR-001/003/004.
