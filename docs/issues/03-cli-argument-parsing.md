# 03 — CLI argument parsing and command capture

## Title
CLI argument parsing and command capture

## Summary
Implement the clap-based CLI from DESIGN §4: global options, subcommands (`doctor`,
`version`, `completions`), and the exact rules for capturing the target command in argv
form vs shell (`-c`) form, including the `--` boundary.

## Context
Getting command capture right is security-relevant (DESIGN §10.3 CLI boundary): dryrun
must never accidentally pass argv-form commands through a shell, and must split its own
flags from the command unambiguously.

## Scope
- In: `src/cli/mod.rs` (clap definitions), `src/cli/args.rs` (typed parsed result), flag
  validation (mutual exclusivity), command capture rules. Subcommand routing stubs for
  `doctor`/`completions` (implemented in 27/34).
- Out: config merge (04), policy build (05), orchestration (26).

## Detailed Requirements
1. Define a clap `Command`/derive struct with every global option in DESIGN §4.2, exact
   long/short names, defaults, and value parsers (sizes like `10MiB`, durations like
   `120s`). Implement size/duration parsers or use a vetted crate from research/02. This
   issue is the **sole owner** of the CLI surface: ALL v1 flags live here, including the
   ones consumed by later issues — `--writable-force` (05), `--fail-on-violation` (05/26),
   `--no-drift-check` (25), `--keep-run-dir` (11/26). Later issues consume the typed fields;
   they MUST NOT introduce new flags. If a later issue needs a flag, it is added here first.
2. Command capture (DESIGN §4.1):
   - Argv form: first non-option token starts the command; all remaining tokens
     (including ones that look like flags) belong to the command. Use clap
     `trailing_var_arg` / `allow_hyphen_values` semantics so `dryrun rm -rf build` yields
     program `rm`, args `["-rf","build"]`.
   - `--` explicitly ends dryrun options; everything after is the command argv.
   - `-c/--command <STR>` shell form: mutually exclusive with argv form. Produces
     `CommandSpec::Shell(String)`.
   - Error (exit 40) if both forms are given, or neither (except for subcommands).
3. Produce `ParsedArgs` (typed): `command: CommandSpec`, all options as typed fields
   (`Option<PathBuf>`, `bool`, parsed sizes/durations), plus `subcommand: Option<Sub>`.
4. Mutually-exclusive / invalid combos → `DryrunError::Config` (exit 40) with a clear
   message: e.g. `--yes` + `--no-execute`; `--json` implies non-interactive (document that
   `--yes`/`--no-execute` still apply).
5. `--no-color`/`NO_COLOR`/non-TTY resolution is captured as an intent here; actual color
   decisions live in the renderer (23) but expose the resolved boolean.
6. Wire `dryrun completions <shell>` to `clap_complete` (generation lives in 34; here just
   the subcommand shape) and `dryrun doctor [--json]` / `dryrun version [--json]`.
7. `--help` output is stable; a golden test is added in 34, but ensure help text is
   authored here (about strings, long help for the network/writable/engine flags).

## Acceptance Criteria
- `dryrun rm -rf build` → `CommandSpec::Argv{program:"rm", args:["-rf","build"]}`.
- `dryrun -- ls -la` → argv `ls`, `["-la"]`.
- `dryrun -c "a && b"` → `CommandSpec::Shell("a && b")`.
- `dryrun -c x foo` and giving both forms → exit 40 with message.
- `--yes --no-execute` → exit 40.
- All DESIGN §4.2 flags parse to the correct typed field with correct defaults (unit
  tests per flag).
- Unknown flag → clap error → exit 40 (mapped, not a panic).

## Validation
- `cargo test cli::` (table-driven capture tests + mutual-exclusion tests).
- Manual: `dryrun --help`, `dryrun rm -rf build --json` parse smoke.

## Dependencies
01, 02.

## Non-goals
No config file reading, no policy construction, no execution.

## Design References
DESIGN §4.1–4.4 (CLI), §10.3 (CLI boundary), §5 (fields consumed later).
