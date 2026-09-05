# ADR-004: Write capture is scoped to the workspace

- Status: Accepted
- Date: 2026-07-10
- Deciders: user (product owner), Fable (design)
- Related: ADR-001, ADR-003, DESIGN.md §Policy model, §Security model

## Context

dryrun must decide which writes it *captures and diffs* (turning them into a reviewable
changeset) versus which writes it *denies*. Capturing writes anywhere on the system would,
on macOS, require heavyweight mechanisms (system extensions) that violate the
no-external-runtime constraint and would give asymmetric guarantees between OSes. The user
selected **workspace-scoped** during clarification.

## Decision

A rehearsal has a **write allowlist** ("the workspace"):

- Default workspace = the current working directory subtree.
- Additional writable roots may be added with `--writable <path>` (repeatable).
- A private, discarded temp directory is always writable and mapped so the command's
  `/tmp` and `$TMPDIR` writes work without polluting the real temp.

Writes inside the workspace are captured (recorded, diffed, discarded after rehearsal).
Writes **outside** the workspace are denied by the sandbox, and each denied attempt is
reported **best-effort** as an "out-of-workspace write attempt" (the kernel primitives
guarantee the *deny*; complete per-attempt enumeration is not guaranteed and is surfaced as
a caveat — see the reporting-fidelity note below). This is still a first-class signal,
because a command trying to modify `~/.zshrc`, `~/.ssh/`, or `/usr/local` is exactly the
kind of surprise the user wants to catch.

**Temp directories** (`/tmp`, `$TMPDIR`): a private, discarded temp is always writable and
mapped so that writes via `$TMPDIR` (and, on the overlay engine, `/tmp` transparently) work
without polluting the real temp. On the copy engines (macOS, portable), a **hard-coded**
absolute write to real `/tmp/...` is treated like any other out-of-workspace write: it is
**denied** (and reported best-effort, per the reporting-fidelity note below; user decision
2026-07-11), not silently allowed to hit the real `/tmp`. Users who need real `/tmp` captured can add it with `--writable /tmp`. This keeps
the guarantee consistent ("only the workspace + private tmp is writable") across engines.

Reads are allowed broadly by default (commands legitimately read config, libraries,
system files). Read confinement **and** sensitive-read *reporting* are both v2 (the v1
primitives do not provide a reliable read-observation channel); v1 makes no claim to report
reads. Redaction still ensures secret file *contents* never appear in reports (§10.4).

### Reporting-fidelity note

Landlock EACCES, netns unreachability, and seccomp `RET_ERRNO` deny the action but do not
provide a guaranteed, complete log of every blocked attempt. v1 therefore reports blocked
writes / network attempts **best-effort**, and every report carries a caveat stating that
the *deny* is enforced by the kernel while the *enumeration* may be incomplete.

## Consequences

Positive:

- Identical, explainable guarantee on macOS and Linux: "we capture writes to your
  workspace and block+report everything else."
- Out-of-workspace write attempts become a headline security signal rather than a silent
  denial.
- Keeps the sandbox implementation within unprivileged, dependency-free primitives on both
  OSes.

Negative / accepted costs:

- A command that legitimately writes outside the workspace (e.g. an installer targeting
  `/usr/local`) will show denied attempts and an incomplete picture unless the user adds
  `--writable`. Documented; the report explains exactly which paths were blocked so the
  user can decide.
- Because the rehearsal runs against a shadow of the workspace, commands using absolute
  paths to the workspace behave differently on copy-based engines vs overlay; see ADR-001
  consequences and the DESIGN.md caveats.

## Alternatives considered

- **Whole-system write capture**: rejected — needs macOS system extensions, asymmetric
  guarantees, heavier and riskier v1.
- **Linux full-FS shadow, macOS workspace-only**: rejected — asymmetric guarantee is
  confusing and doubles the documentation/test burden.
