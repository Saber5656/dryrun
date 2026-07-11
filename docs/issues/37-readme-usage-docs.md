# 37 — User-facing README and usage docs

## Title
User-facing README and usage docs

## Summary
Write the public `README.md`: what dryrun is, install, quickstart, the safety model in
plain language (network deny, workspace-only writes, fail-closed, prediction-not-guarantee),
engine/OS support, `doctor`, and the JSON contract for automation (DESIGN §1, §10.7).

## Context
The README is the front door for an OSS project. It must set correct expectations about the
tool's guarantees and limits (especially "the diff is a prediction" and the engine
differences) so users trust it appropriately.

## Scope
- In: `README.md` (replacing the current one-line file), usage examples, a short
  troubleshooting section pointing to `dryrun doctor`.
- Out: SECURITY/CONTRIBUTING (35); generated man/completions (34).

## Detailed Requirements
1. Sections:
   - one-paragraph pitch (keep the original concept line as the tagline),
   - install (from GitHub Releases with checksum verification; `cargo install` note),
   - quickstart: `dryrun rm -rf build`, reading the diff, approving; `-c` shell form;
     `--json` for automation,
   - safety model in plain language: network denied by default (`--allow-network`), writes
     confined to and captured within the workspace (`--writable` to extend, out-of-
     workspace attempts are reported), fail-closed sandbox, **the diff is a prediction, not
     a guarantee** (ADR-005) and why (two executions, non-determinism),
   - platform/engine support table (macOS seatbelt, Linux overlay/portable) with the
     Ubuntu-24.04 userns note and a pointer to `dryrun doctor`,
   - exit codes table (link DESIGN §4.3) for scripters/agents,
   - JSON report pointer (link `docs/report.schema.json`) for automation,
   - security disclosure pointer to `SECURITY.md`,
   - license (per ADR-006), contributing pointer.
2. Every guarantee statement must be accurate to the engine matrix (DESIGN §3.5) — do not
   overpromise (e.g. blocked-write enumeration is best-effort on some engines; say so).
3. Include a short "limitations" subsection mirroring DESIGN §2.2 non-goals (no Windows, no
   read confinement, no rollback, servers/TUIs caveat).
4. Keep examples copy-pasteable and OS-annotated where behavior differs.

## Acceptance Criteria
- README covers install, quickstart, safety model, engine/OS support, exit codes, JSON
  pointer, security + license, and limitations.
- No guarantee in the README contradicts DESIGN §3.5 or §2.2.
- Links to `docs/report.schema.json`, `SECURITY.md`, DESIGN exit-code section resolve.

## Validation
- Markdown link check (CI optional); manual review against DESIGN §3.5/§2.2 for accuracy;
  Codex per-issue review.

## Dependencies
03 (flags/exit surface), 26 (working end-to-end to document real behavior). Content
references 34/35/36 outputs.

## Non-goals
No API docs generation; no marketing site; no SECURITY/CONTRIBUTING content (35).

## Design References
DESIGN §1 (overview), §2.2 (non-goals/limitations), §3.5 (engine matrix), §4.3 (exit codes),
§10.7 (secure defaults), ADR-005/006.
