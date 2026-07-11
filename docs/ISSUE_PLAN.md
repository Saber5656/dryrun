# dryrun — Issue Plan (v1)

Version: 2026-07-10
Source of truth: this file + `docs/DESIGN.md` + `docs/issues/*.md`.
GitHub Issues are **derived** from `docs/issues/NN-*.md`. If they disagree, the repo docs
win; update docs first, then reconcile the GitHub Issue.

## v1 completion statement

v1 is **complete** when every issue below (01–38) is implemented, validated per its
Acceptance Criteria, and merged, such that:

> On macOS (APFS) and Linux (both userns-enabled and userns-restricted configurations), a
> user can run `dryrun <command>`, have it rehearsed in an OS-native, fail-closed sandbox
> with external network denied by default (local IPC allowed) and writes confined to (and
> captured within) the workspace, review a human diff **or** a versioned JSON report of the
> resulting ChangeSet (with out-of-workspace writes and network attempts kernel-denied and
> reported best-effort under an explicit fidelity caveat), and then approve a real
> re-execution guarded by a drift check — with the tool shipping as a single binary through
> a hardened, reproducible, security-documented OSS release.

Anything not covered by 01–38 and not listed in "Deferred v2" or "Known unknowns" is a
gap; add an issue rather than leaving behavior in prose.

## Issue list (recommended execution order)

| NN | Title | File |
|----|-------|------|
| 01 | Project scaffold, toolchain, lints, module stubs | [01-project-scaffold.md](issues/01-project-scaffold.md) |
| 02 | Error types and exit-code mapping | [02-error-and-exit-codes.md](issues/02-error-and-exit-codes.md) |
| 03 | CLI argument parsing and command capture | [03-cli-argument-parsing.md](issues/03-cli-argument-parsing.md) |
| 04 | Config file loading and precedence | [04-config-loading.md](issues/04-config-loading.md) |
| 05 | Policy model and path containment | [05-policy-and-path-containment.md](issues/05-policy-and-path-containment.md) |
| 06 | SandboxEngine trait and core run types | [06-engine-trait-core-types.md](issues/06-engine-trait-core-types.md) |
| 07 | Engine selection and capability probe | [07-engine-selection-probe.md](issues/07-engine-selection-probe.md) |
| 08 | ChangeSet data model | [08-changeset-data-model.md](issues/08-changeset-data-model.md) |
| 09 | Bounded stdio capture | [09-bounded-stdio-capture.md](issues/09-bounded-stdio-capture.md) |
| 10 | Resource monitor (walltime, disk, inodes) | [10-resource-monitor.md](issues/10-resource-monitor.md) |
| 11 | Shadow builder and pre-run manifest | [11-shadow-builder-manifest.md](issues/11-shadow-builder-manifest.md) |
| 12 | Tree-diff changeset scanner (copy engines) | [12-treediff-scanner.md](issues/12-treediff-scanner.md) |
| 13 | Landlock ruleset builder | [13-landlock-ruleset-builder.md](issues/13-landlock-ruleset-builder.md) |
| 14 | Diff computation, classification, redaction | [14-diff-classify-redaction.md](issues/14-diff-classify-redaction.md) |
| 15 | macOS Seatbelt/clonefile FFI | [15-macos-ffi.md](issues/15-macos-ffi.md) |
| 16 | macOS SBPL profile builder | [16-macos-sbpl-profile.md](issues/16-macos-sbpl-profile.md) |
| 17 | macos-seatbelt engine assembly | [17-macos-seatbelt-engine.md](issues/17-macos-seatbelt-engine.md) |
| 18 | Linux seccomp socket-family deny filter | [18-linux-seccomp-filter.md](issues/18-linux-seccomp-filter.md) |
| 19 | linux-portable engine | [19-linux-portable-engine.md](issues/19-linux-portable-engine.md) |
| 20 | Linux namespace and overlayfs helpers | [20-linux-namespace-overlay-helpers.md](issues/20-linux-namespace-overlay-helpers.md) |
| 21 | linux-overlay engine and whiteout scan | [21-linux-overlay-engine.md](issues/21-linux-overlay-engine.md) |
| 22 | JSON report emitter and committed schema | [22-json-report-schema.md](issues/22-json-report-schema.md) |
| 23 | Human diff renderer with ANSI sanitization | [23-human-renderer.md](issues/23-human-renderer.md) |
| 24 | Decision gate and interactive prompt | [24-decision-gate.md](issues/24-decision-gate.md) |
| 25 | Drift check and real execution | [25-drift-check-real-exec.md](issues/25-drift-check-real-exec.md) |
| 26 | Orchestrator and run state machine | [26-orchestrator-state-machine.md](issues/26-orchestrator-state-machine.md) |
| 27 | `dryrun doctor` capability report | [27-doctor-command.md](issues/27-doctor-command.md) |
| 28 | Path/symlink/TOCTOU escape regression tests | [28-escape-regression-tests.md](issues/28-escape-regression-tests.md) |
| 29 | Resource-exhaustion tests | [29-resource-exhaustion-tests.md](issues/29-resource-exhaustion-tests.md) |
| 30 | Redaction and ANSI-injection tests | [30-redaction-ansi-tests.md](issues/30-redaction-ansi-tests.md) |
| 31 | Network default-deny end-to-end tests | [31-network-deny-e2e-tests.md](issues/31-network-deny-e2e-tests.md) |
| 32 | cargo-deny and supply-chain gates | [32-cargo-deny-supply-chain.md](issues/32-cargo-deny-supply-chain.md) |
| 33 | CI matrix (macOS + Linux userns on/off) | [33-ci-matrix.md](issues/33-ci-matrix.md) |
| 34 | Man page, completions, help golden tests | [34-manpage-completions.md](issues/34-manpage-completions.md) |
| 35 | Security and contribution documentation | [35-security-contrib-docs.md](issues/35-security-contrib-docs.md) |
| 36 | Release pipeline, licenses, checksums | [36-release-pipeline.md](issues/36-release-pipeline.md) |
| 37 | User-facing README and usage docs | [37-readme-usage-docs.md](issues/37-readme-usage-docs.md) |
| 38 | AppArmor userns profile for restricted distros | [38-apparmor-profile.md](issues/38-apparmor-profile.md) |

## Dependency table

"Depends on" = must be merged first. Cross-check with each issue's Dependencies section.

| NN | Depends on |
|----|------------|
| 01 | — |
| 02 | 01 |
| 03 | 01, 02 |
| 04 | 01, 02 |
| 05 | 01, 02, 03, 04 |
| 06 | 01, 02 |
| 07 | 05, 06 |
| 08 | 01 |
| 09 | 06 |
| 10 | 06 |
| 11 | 05, 06, 15 |
| 12 | 08, 11 |
| 13 | 05, 06 |
| 14 | 08 |
| 15 | 01 |
| 16 | 05 |
| 17 | 05, 06, 09, 10, 11, 12, 15, 16 |
| 18 | 05, 06 |
| 19 | 05, 06, 09, 10, 11, 12, 13, 18 |
| 20 | 05, 06 |
| 21 | 05, 06, 08, 09, 10, 13, 20 |
| 22 | 02, 05, 06, 07, 08 |
| 23 | 08, 14 |
| 24 | 05, 22, 23 |
| 25 | 05, 06, 08, 11 |
| 26 | 02, 03, 04, 05, 06, 07, 08, 22, 23, 24, 25, + one engine (17 \| 19 \| 21) |
| 27 | 07 |
| 28 | 26, + one engine (17 \| 19 \| 21) |
| 29 | 10, 26 |
| 30 | 14, 23 |
| 31 | 26, + one engine (17 \| 19 \| 21) |
| 32 | 01 |
| 33 | 26, 32 |
| 34 | 03 |
| 35 | — (docs; content depends on DESIGN.md) |
| 36 | 32, 33, 34, 38 (ADR-006 license confirmed) |
| 37 | 03, 26 |
| 38 | 01, 20, 21 (validated against overlay engine) |

## Implementation waves

Each wave can be worked mostly in parallel internally; a wave starts when its
dependencies from prior waves are merged.

- **Wave 0 — Foundation**: 01, 02.
- **Wave 1 — Frontend + policy**: 03, 04 (before 05), then 05. (05 consumes 03/04's types.)
- **Wave 2 — Core contracts**: 06, 07, 08, 09, 10.
- **Wave 3 — Shared building blocks**: 11, 12, 13, 14, plus 15 (macOS FFI depends only on
  01, so it can land here; 11's macOS reflink path needs it — see 11's dependency note).
- **Wave 4 — Engines**:
  - macOS: 16, 17 (15 landed in Wave 3).
  - Linux portable: 18, 19.
  - Linux overlay: 20, 21.
- **Wave 5 — Output + flow**: 22, 23, 24, 25, 26, 27.
- **Wave 6 — Security & behavior tests**: 28, 29, 30, 31.
- **Wave 7 — OSS release baseline**: 32, 33, 34, 35, 36, 37, 38.

Minimum end-to-end vertical slice (earliest demoable): 01→02→03→04→05→06→07→08→09→10→
(18,19 for the most portable engine)→22→23→24→25→26. Completing that lets `dryrun` work on
Linux via the portable engine before the overlay/macOS engines land; use it as an
integration checkpoint.

## Coverage: DESIGN.md section → issue(s)

| DESIGN.md section | Covered by |
|---|---|
| 1 Product overview | 37 (README), 35 (docs) |
| 2 Scope / non-goals / v2 / unknowns | ISSUE_PLAN (this file), 35 |
| 3.1 Process model | 26 |
| 3.2 Module layout | 01 |
| 3.3 Engine trait | 06 |
| 3.4 Engine selection | 07 |
| 3.4 Remediation (Ubuntu userns / AppArmor profile) | 27, 38 |
| 3.5 Guarantee matrix | 07 (report), 17, 19, 21 |
| 4.1–4.2 CLI + options | 03 |
| 4.3 Exit codes | 02 |
| 4.4 Decision gate | 24 |
| 5 Policy model | 05 |
| 5.1 Path containment | 05, 28 |
| 5.2 Precedence | 04 |
| 6 ChangeSet model | 08 |
| 6.1 Change detection (overlay) | 21 |
| 6.1 Change detection (copy) | 11, 12 |
| 6.2 Determinism | 08, 12, 22 |
| 7 JSON report schema | 22 |
| 8 Run state machine | 26 |
| 9 Human rendering | 23 |
| 10.1–10.2 Threat model | 05, 17, 19, 21, 28, 29, 30, 31, 35 |
| 10.3 Boundary expectations | 03 (CLI), 04 (config), 12 (scan), 18/20/21 (network), 15 (FFI), 13 (real-exec via 25) |
| 10.3 Reporting-fidelity (best-effort + caveat) | 16, 17, 18, 19, 21 (engines emit caveats), 08 (caveats field) |
| 10.4 Secret handling | 14, 30 |
| 10.5 Abuse cases | 05 (writable deny), 04 (config narrow-only), 35 (docs) |
| 10.6 Supply chain | 32, 33, 36 |
| 10.7 Secure defaults | 05, 18, 19, 21, 17, 24, 13 |
| 11 Failure modes | 02, 07, 10, 25, 26 |
| 12 Config file | 04 |
| 13 Run-dir layout | 11, 20, 26 |
| 14 Observability | 26, 27 |
| 15 Testing strategy | 28, 29, 30, 31, 33, 34 |
| Network default-deny (ADR-003) | 16, 18, 20, 21, 31 |
| Workspace scope (ADR-004) | 05, 11, 12, 13, 17, 19, 21 |
| Approve-then-reexecute (ADR-005) | 24, 25 |

Every DESIGN.md section maps to at least one issue. No product behavior lives only in
prose.

## Whole-product validation strategy

1. **Per-issue AC**: each issue carries concrete Acceptance Criteria + Validation
   commands. An issue is done only when its validation passes in CI.
2. **Engine conformance suite** (28, 31 + engine issues): one shared test table
   ("create/modify/delete/rename/mode/symlink/out-of-workspace/network") is run against
   every available engine on its OS; the guarantee matrix (DESIGN §3.5) is asserted, and
   caveats are asserted present in reports.
3. **Fail-closed assertions**: forcing an unusable engine, breaking sandbox init, and
   removing userns all must error (42/40/41), never run unsandboxed. Explicit tests.
4. **Security regression suite** (28, 29, 30): escapes, exhaustion, redaction, ANSI
   injection — each a named test that must stay green.
5. **CI matrix** (33): macOS (APFS) + Linux userns-enabled + Linux userns-restricted
   (AppArmor) so both the overlay engine and the portable fallback are exercised on every
   PR.
6. **Golden/snapshot** (22, 23, 34): JSON schema, help text, man page, SBPL profile,
   sample reports are snapshot-tested to catch unintended output/contract drift.
7. **Supply-chain gate** (32): `cargo deny check` + `--locked` build on every PR.
8. **Determinism check** (08/12/22): a rehearsal of a fixed fixture yields byte-stable
   JSON (minus the `meta` timestamp block).

## Deferred to v2 (not in v1 scope; see DESIGN §2.3)

- Read confinement / secret-read denial **and** read *reporting* (v1 tracks no reads; the
  v1 primitives provide no reliable read-observation channel — reads are neither blocked nor
  reported in v1; contents are still redacted from reports).
- Change promotion (apply captured changes without re-exec) for file-only commands.
- Interactive network-allow prompt on first blocked socket.
- `ratatui` TUI review surface.
- FreeBSD engine; Seatbelt-replacement engine.
- Multi-step script "plan" rehearsal with per-step diffs.
- Named/org-shared policy profiles.
- Content-addressed rehearsal cache.
- Windows support.

## Known unknowns (may create new issues during implementation)

Mirror of DESIGN §2.4; each may spawn a follow-up issue:

- U1 Landlock ABI variance across kernels → possibly per-ABI code paths (would extend 13).
- U2 macOS Tahoe+ SBPL drift → may add profile entries / more golden tests (extends 16).
- U3 Seatbelt removal → would add a replacement engine (new issue, isolated by trait).
- U4 overlayfs whiteout/opaque encoding across kernels under `userxattr` → may extend 21.
- U5 shells reading unexpected sysctls/paths → extends 16.
- U6 `clonefile` on exotic filesystems → extends 11.
- U7 reliable "blocked-by-network" detection heuristic → may add a signature-list issue.
- U8 symlink/hardlink rename reconstruction on copy engines → may extend 12.

When implementation hits one of these, add a `docs/issues/NN-*.md`, update this plan and
the coverage table, then (if warranted) create the GitHub Issue — docs first.
