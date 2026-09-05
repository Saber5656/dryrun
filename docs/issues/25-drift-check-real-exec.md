# 25 — Drift check and real execution

## Title
Drift check and real execution

## Summary
After approval, verify that workspace inputs the rehearsal touched have not changed since
rehearsal (drift check), warn/abort on drift, then execute the same command for real
(unsandboxed, normal network) streaming output (DESIGN §8, ADR-005).

## Context
Rehearsal and real run are two executions and can diverge under non-determinism or if files
changed in between (ADR-005). The drift check reduces surprise before the irreversible real
run.

## Scope
- In: `src/execreal/mod.rs` — `drift_check(policy, changeset, manifest) -> DriftReport` and
  `run_real(policy, cmd, io) -> ExitStatus`.
- Out: the gate (24); sandbox engines.

## Detailed Requirements
1. `drift_check`: the tool does NOT track reads in v1, so the drift set is defined
   explicitly and conservatively as **the workspace paths that appear in the ChangeSet**
   (created/modified/deleted/renamed/mode/type/symlink entries) — i.e. the inputs/outputs
   the rehearsal actually touched. For each such path, take its pre-rehearsal `Meta` from
   the manifest (11) and re-stat + (size/mtime changed ⇒ re-hash) the current real file;
   flag any that differ from the manifest. Produce `DriftReport { changed: Vec<PathBuf>,
   any: bool }`. This is bounded by the changeset size (already capped by `max_files`), so
   it is cheap.
   - Explicitly documented v1 limitation (add to report caveats + README): drift detection
     covers only paths the rehearsal changed; it does NOT detect that an untouched input the
     real run will read has changed, nor any out-of-workspace read drift (reads are not
     tracked — N2). This is a best-effort guard, not a guarantee.
2. On drift:
   - TTY + `Ask`/`Yes`: warn, list changed paths; if `Ask`, re-confirm (reuse 24's
     prompt); if `--yes`, proceed but print a clear note that inputs changed since preview.
   - Provide `--no-drift-check` to skip (documented, discouraged) and record it in output.
3. `run_real`: execute the exact same `CommandSpec` (argv or `sh -c`) in the **real**
   environment — real cwd (the actual workspace, not the shadow), inherited env, normal
   filesystem, normal network, no sandbox. Stream stdout/stderr live (tee optional; no byte
   cap needed for the real run, but keep the same signal handling). Return the child's
   `ExitStatus`.
4. Map real-run non-zero → exit 50 (`REAL_RUN_FAILED`); success → 0. Signals handled
   (SIGINT forwarded/terminates as expected).
5. The real run is only reachable after an Approve decision (24) — assert this ordering in
   the orchestrator (26); this module must not run real without an approval flag passed in.

## Acceptance Criteria
- With no drift, approval proceeds straight to the real run.
- If a touched input changes between rehearsal and approval, drift is detected and (TTY)
  re-confirmation is required; `--yes` proceeds with a printed note.
- The real run executes the identical command in the real workspace and returns the true
  exit status (0 → exit 0; non-zero → exit 50).
- `run_real` cannot be invoked without an approval token/flag (compile- or assert-enforced).

## Validation
- `cargo test execreal::` : drift detection on a mutated fixture; identical-command real
  execution against a temp workspace verifying real effects occur; exit-code mapping.

## Dependencies
05, 06, 08, 11. (Drift compares against the pre-run `Manifest` type defined in 11.)

## Non-goals
No rollback/undo (N8); no change promotion (ADR-005 rejects for v1); no sandboxing.

## Design References
DESIGN §8 (state machine, drift), §4.3 (exit 50), ADR-005.
