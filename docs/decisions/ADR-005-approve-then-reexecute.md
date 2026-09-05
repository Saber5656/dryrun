# ADR-005: Approval re-executes the command (rehearsal is not promoted)

- Status: Accepted
- Date: 2026-07-10
- Deciders: user (product owner), Fable (design)
- Related: ADR-003, ADR-004, DESIGN.md §Run state machine

## Context

After the user reviews the diff, dryrun must produce the *real* effect. Two models were
possible: (a) re-run the same command unsandboxed after approval, or (b) "promote" the
captured shadow changes onto the real filesystem without re-running. The user selected
**re-execute after approval** during clarification.

## Decision

On approval, dryrun executes the **same command** in the real environment (normal
filesystem, normal network), streaming its output. The captured shadow from the rehearsal
is used only for preview and is discarded.

Rationale:

- Model (b) is unsafe for any command whose effects are not purely file writes: network
  calls, process side effects, and daemon interactions cannot be "promoted." Promotion
  would also need a conflict-detection + rollback engine (what if the real files changed
  since rehearsal?) that is the single hardest, most dangerous component to get right.
- Model (a) matches user intuition ("preview, then actually run it") and keeps v1's
  risk surface bounded.

Because rehearsal and real run are two executions, results can differ due to
**non-determinism** (clock, randomness, network responses, concurrent processes, files
changed between the two runs). dryrun:

- states plainly that the diff is a *prediction*, not a guarantee;
- before the real run, re-checks that workspace inputs it relied on have not changed since
  rehearsal (a cheap mtime/size/hash check on touched files) and warns on drift;
- offers `--yes` to auto-approve (for humans who trust the preview) and a JSON contract
  (for agent flows) so approval can be automated by the caller.

## Consequences

Positive:

- Simple, predictable mental model; no rollback engine in v1.
- Works for every command type, not just file-write-only commands.

Negative / accepted costs:

- The rehearsal's diff can diverge from the real run's effect under non-determinism;
  mitigated by the drift check and explicit "prediction, not guarantee" framing.
- The command runs twice (once sandboxed, once real). For expensive commands this is extra
  cost; `--no-execute`/preview-only mode exists for users who only want the diff.

## Alternatives considered

- **Promote captured changes (model b)**: rejected for v1 — unsafe for non-file effects,
  requires a conflict/rollback engine, high blast radius. Recorded as a possible bounded
  v2 exploration only for file-only commands with explicit conflict detection.
- **Preview-only, never execute**: available as a mode (`--no-execute`), but not the
  default, per the user's "re-execute after approval" choice.
