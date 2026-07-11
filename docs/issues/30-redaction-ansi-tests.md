# 30 — Redaction and ANSI-injection tests

## Title
Redaction and ANSI-injection tests

## Summary
Prove secrets never leak into reports and that attacker-controlled paths/diff content cannot
inject terminal control sequences into the human renderer (DESIGN §10.2 T6/T7, §10.4).

## Context
Redaction (14) and sanitization (23) are security controls; these tests guarantee they hold
against crafted inputs and stay green as those modules evolve.

## Scope
- In: `tests/redaction.rs`, `tests/ansi_injection.rs`.
- Out: the redaction/sanitization impls (14/23).

## Detailed Requirements
1. Redaction:
   - Rehearse a command that writes a fake secret to a redaction-matching path
     (`.aws/credentials`, `id_rsa`, `.env`) and to a file containing a `-----BEGIN PRIVATE
     KEY-----` block. Serialize both the JSON report and the human output and assert the
     secret string is ABSENT from both, while the file still appears as a change with
     metadata + `Redacted`.
   - Assert redaction-on-by-default; with redaction disabled via config, the diff appears
     (documents the toggle) — but the default path is the security-critical one.
2. ANSI/control injection:
   - Create files whose **paths** and **contents** contain ESC sequences, cursor moves,
     title-set sequences, and very long lines. Render human output and assert: no raw
     `0x1b` byte appears; lines are length-capped; our own styling is intact.
   - Assert the JSON report stores the raw path faithfully (JSON is not a terminal) but the
     human renderer neutralizes it.
3. Cross-check: scan the entire serialized human+JSON output bytes for the injected secret
   and for raw ESC (human only) programmatically.

## Acceptance Criteria
- Injected secret values never appear in human or JSON output under default settings.
- Human output for malicious paths/contents contains no raw `0x1b`; lines are capped.
- Redacted files still show as changes (metadata only).

## Validation
- `cargo test --test redaction --test ansi_injection`, wired into CI (33).

## Dependencies
14, 23.

## Non-goals
No escape (28); no resource (29); no network (31).

## Design References
DESIGN §10.2 T6/T7, §10.4 (secret handling), §9 (rendering), §15.
