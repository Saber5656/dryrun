# ADR-006: Open-source license and public posture

- Status: Accepted (license confirmed by user 2026-07-11: MIT OR Apache-2.0)
- Date: 2026-07-10 (accepted 2026-07-11)
- Deciders: user (product owner), Fable (design)
- Related: task prompt ("intended to become an open-source project"), DESIGN.md §Release & supply-chain

## Context

The repository is intended to be released publicly as open source. The license choice
affects `Cargo.toml` metadata, `LICENSE*` files, `SPDX-License-Identifier` headers, the
README badge, and contributor expectations, so it must be pinned before the release-prep
issues run.

## Decision (proposed)

Dual-license under **MIT OR Apache-2.0**, the Rust ecosystem default.

- Ship `LICENSE-MIT` and `LICENSE-APACHE`; set `license = "MIT OR Apache-2.0"` in
  `Cargo.toml`.
- Apache-2.0 provides an explicit patent grant; MIT provides maximal permissiveness and
  familiarity. Offering both is the ecosystem norm and maximizes downstream adoption.
- Copyright holder line and year are filled in from the user's confirmed identity during
  the release-prep issue (do not invent a legal name).

## Consequences

Positive:

- Matches Rust community expectations; frictionless dependency adoption; patent grant
  present.

Negative / accepted costs:

- Two license files to keep in sync (standard, low cost).

## Resolution

The user confirmed **A. MIT OR Apache-2.0** on 2026-07-11. This ADR is Accepted. Issue 01
sets `license = "MIT OR Apache-2.0"`; issue 36 ships `LICENSE-MIT` and `LICENSE-APACHE`.
The copyright holder line/year is still filled in from the user's confirmed identity during
the release-prep issue (do not invent a legal name).
