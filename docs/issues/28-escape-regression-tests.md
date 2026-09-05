# 28 — Path/symlink/TOCTOU escape regression tests

## Title
Path/symlink/TOCTOU escape regression tests

## Summary
Add a security regression suite proving the rehearsal cannot write outside the workspace via
symlinks, `..` traversal, absolute paths, or TOCTOU races, on every available engine
(DESIGN §10.2 T1/T4).

## Context
Write confinement is the product's core guarantee; these are the tests that would catch a
regression that silently weakens it. They must run on each engine on its OS.

## Scope
- In: `tests/escape_*.rs` (integration), a shared helper table run per available engine.
- Out: the engines themselves (17/19/21); resource tests (29).

## Detailed Requirements
1. Build a shared harness that, given an engine, runs a command in a temp workspace and
   asserts the real filesystem outside the workspace is unchanged and (where enumerable) the
   attempt is reported in `blocked_writes`.
2. Test targets MUST be a temp directory the test user can normally write to but that lies
   OUTSIDE the workspace (e.g. a second `tempfile::tempdir()` called `outside/`). Do NOT use
   `/etc/...` as the probe target: a write there fails for ordinary users even with no
   sandbox, so the test would pass for the wrong reason. Each scenario asserts the outside
   target's hash/stat is unchanged (proving the sandbox, not filesystem permissions,
   blocked it). Scenarios (each a named test):
   - symlink inside workspace → `<outside>/probe`; command writes through it. Assert
     `<outside>/probe` is NOT created.
   - `../probe.txt` relative write above the workspace (into `<outside>`). Assert not
     created.
   - absolute write to `<outside>/probe`. Assert not created.
   - symlinked directory whose target is `<outside>`; writing a file inside it. Assert not
     created.
   - hard-coded `/tmp/dryrun_probe_<rand>` write on copy engines → denied+reported per
     ADR-004 (assert real `/tmp/dryrun_probe_<rand>` is NOT created).
   - TOCTOU: a background thread flips a path to a symlink pointing at `<outside>` during the
     run (best-effort; assert the kernel enforcement, not the userspace check, prevents the
     escape).
   - hardlink to a file in `<outside>` then write. Assert the outside file unchanged.
3. Each scenario asserts: (a) outside target unchanged (hash/stat before==after), (b) exit
   code sane, (c) where the engine can enumerate, the attempt appears in the report; else the
   completeness caveat is present.
4. Tests are os/engine-gated (skip with a message where an engine is unavailable) so the
   suite is green on both CI OSes while covering whatever engines exist there.

## Acceptance Criteria
- All scenarios pass on the portable engine (Linux) and the overlay engine (Linux, userns)
  and the seatbelt engine (macOS).
- A deliberately weakened confinement (e.g. commenting out Landlock apply) makes at least
  one scenario fail — i.e. the tests actually detect regressions (verify once during
  development, then restore).

## Validation
- `cargo test --test escape_symlink --test escape_dotdot --test escape_absolute ...`
  (os-gated). Wired into CI (33) on every PR.

## Dependencies
26 and at least one engine (17/19/21). Runs against all engines present on the CI OS.

## Non-goals
No resource-exhaustion (29); no redaction (30); no network (31).

## Design References
DESIGN §10.2 T1/T4, §5.1 (containment), §15 (security regression suite), ADR-004.
