# 35 — Security and contribution documentation

## Title
Security and contribution documentation

## Summary
Author the OSS governance/security docs: `SECURITY.md` (disclosure policy + threat-model
summary), `CONTRIBUTING.md` (build/test/dep policy), `CODE_OF_CONDUCT.md`, and GitHub
issue/PR templates (DESIGN §10, §10.6).

## Context
A publicly released security tool needs a clear vulnerability-disclosure channel and
contributor expectations. These derive from the DESIGN security model; they are docs, not
code, so they can be written early.

## Scope
- In: `SECURITY.md`, `CONTRIBUTING.md`, `CODE_OF_CONDUCT.md`, `.github/ISSUE_TEMPLATE/*`,
  `.github/pull_request_template.md`.
- Out: README (37); release/signing (36).

## Detailed Requirements
1. `SECURITY.md`:
   - private disclosure channel (a security contact/email or GitHub private advisory —
     the actual address is provided by the user; leave a clearly-marked placeholder and
     flag it as a required user input, do not invent one),
   - supported versions policy, response-time expectations,
   - a concise summary of the threat model + trust boundaries (link DESIGN §10),
   - explicit statement that `--allow-network` and `--writable-force` widen the trust
     surface and are user-opt-in.
2. `CONTRIBUTING.md`:
   - toolchain/setup, `cargo fmt`/`clippy`/`test`/`deny` gates (32), the `--locked` rule,
   - dependency policy (research/02): justify new deps, keep `unsafe` in named OS modules
     with `// SAFETY:`,
   - how to run engine tests per OS and the userns on/off note (33),
   - the "docs are source of truth; update `docs/` before code/issues" rule from the
     project's process.
3. `CODE_OF_CONDUCT.md`: adopt a standard (e.g. Contributor Covenant) with the contact
   placeholder flagged for the user.
4. Templates: a bug template that asks for OS/kernel, `dryrun doctor --json` output, and
   engine used; a security note pointing reporters to `SECURITY.md` (not public issues);
   a PR template with a checklist (tests, docs updated, `cargo deny` clean, security impact
   considered).
5. Mark every place needing a real contact/legal detail as a `TODO(user)` so the release
   issue (36) can require them.

## Acceptance Criteria
- All five artifacts exist, internally consistent, and link to DESIGN §10 / research/02.
- No invented contact details; user-input placeholders are clearly flagged.
- PR template checklist includes tests, docs, `cargo deny`, and security impact.

## Validation
- Markdown lint / link check in CI (optional); manual review that placeholders are flagged.
- Reviewed in the Codex per-issue review (project process).

## Dependencies
None (content derives from DESIGN.md). Best done alongside 36.

## Non-goals
No code; no README (37); no signing keys (user-provided, 36).

## Design References
DESIGN §10 (security model), §10.6 (supply chain), ADR-006 (license/posture).
