# 23 — Human diff renderer with ANSI sanitization

## Title
Human diff renderer with ANSI sanitization

## Summary
Render a `ChangeSet` as a colored, skimmable, paged human report (DESIGN §9), with strict
sanitization of attacker-controlled paths/diff text so a hostile command cannot spoof the
UI via ANSI/control sequences (DESIGN §10.2 T6).

## Context
The human diff is the headline experience for the primary user (DESIGN §1.3). It is also a
security surface: file paths and diff contents come from the rehearsed (untrusted) command
and must be neutralized before printing.

## Scope
- In: `src/report/human.rs` — header, summary line, per-file rendering, blocked/network
  sections, color + paging + sanitization.
- Out: JSON (22); the prompt itself (24).

## Detailed Requirements
1. Header: engine + selection caveats, command, workspace, network mode, and any degraded/
   force-writable warnings (DESIGN §9).
2. Summary line with counts (created/modified/deleted/renamed/mode + blocked + network),
   using clear symbols and colors.
3. Per file: badge (`A`/`M`/`D`/`R`/`chmod`/`type`), path, then for `Diff::Text` a unified
   diff (green add / red remove) with context; `Binary`/`TooLarge` show size deltas;
   `Redacted` shows `‹redacted›` and metadata only. Renamed shows `from → to`.
4. **Sanitization (security)**: before printing any command-derived string (paths, diff
   lines), strip or escape ESC (`0x1b`) and other C0 control chars except `\n`/`\t`;
   cap per-line length (e.g. 4 KiB, ellipsize) to prevent terminal-spamming; never pass raw
   attacker bytes to the terminal. Our own styling ANSI is added after sanitization.
5. Blocked writes and network attempts get prominent, separately-headed sections (these are
   the "pay attention" signals) with a distinct color.
6. Color policy: respect `--no-color`, `NO_COLOR`, and non-TTY (from 03's resolved bool);
   use `anstyle`/`anstream` (research/02). Paging: pipe through `$PAGER` (default `less -R`)
   when TTY + long, unless `--no-pager`; handle pager-absent gracefully.
7. Deterministic text output when color is off (snapshot-testable).

## Acceptance Criteria
- A mixed changeset renders header, summary, per-file diffs, and blocked/network sections
  correctly (snapshot with color off).
- A path containing raw ESC/control bytes is sanitized — the rendered output contains no
  raw `0x1b` (asserted by scanning bytes).
- An extremely long diff line is capped/ellipsized.
- `NO_COLOR`/non-TTY disables ANSI; `--no-pager` skips paging; missing `$PAGER` does not
  crash.
- `Redacted` files never show contents.

## Validation
- `cargo test report::human` (snapshot with color off + a sanitization test asserting no
  `0x1b` bytes in output for a malicious path fixture). Deeper injection tests in 30.

## Dependencies
08, 14.

## Non-goals
No prompting/decision (24); no JSON (22).

## Design References
DESIGN §9 (rendering), §10.2 T6 (UI spoofing), §10.4 (redaction display).
