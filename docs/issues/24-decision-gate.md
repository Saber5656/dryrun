# 24 — Decision gate and interactive prompt

## Title
Decision gate and interactive prompt

## Summary
Implement the approve/deny gate after rendering (DESIGN §4.4, §8): the interactive TTY
prompt, the `--yes`/`--no-execute` short-circuits, the JSON/non-interactive behavior, and
the safe default (no real run without an explicit approval channel).

## Context
This is the safety hinge between rehearsal and real execution (DESIGN §10.2 T8): the user
must not approve thinking they are still in preview, and non-interactive contexts must
default to not running.

## Scope
- In: the decision logic (in `src/cli/run.rs` or `src/report/mod.rs`): `decide(policy,
  ttyness, report) -> Decision { Approve | Deny | NoChannel }`.
- Out: drift check + real exec (25); rendering (23).

## Detailed Requirements
1. Precedence:
   - `execute_after == Never` (`--no-execute`) → `Deny`-as-preview (exit OK, no real run).
   - `execute_after == Yes` (`--yes`) → `Approve` (still runs the drift check in 25).
   - `--json` or not a TTY, and not `--yes` → `NoChannel`: do not run for real; exit OK
     with a note ("no approval channel; re-run with --yes to execute" or the JSON caller
     decides). Never auto-run.
   - TTY + `Ask` → prompt.
2. Interactive prompt (DESIGN §4.4): `Apply this for real? [y]es / [N]o / [d]etails /
   [q]uit:`. Default (Enter/EOF) = No. `d` re-renders full detail (re-open pager) then
   re-prompts. `y` → Approve; `N`/`q`/EOF → Deny (exit 20).
3. The prompt text must make the real run unmistakable and echo `--allow-network` if set
   ("This will run for real WITH network access.") to prevent T8 confusion.
4. Read from the controlling TTY, not stdin-pipe, where possible (so a piped command's
   stdin doesn't accidentally answer the prompt); if only piped stdin exists, treat as
   NoChannel.
5. Map decisions to the state machine (DESIGN §8): Approve→DRIFT_CHECK, Deny→DONE(20 or OK
   for --no-execute), NoChannel→DONE(OK with note).

## Acceptance Criteria
- `--no-execute` never runs real; exits OK.
- `--yes` approves (proceeds to drift check).
- `--json`/non-TTY without `--yes` → NoChannel, no real run, exit OK.
- TTY prompt: `y`→approve, Enter/`N`/EOF→deny (exit 20), `d`→re-render then re-prompt.
- With `--allow-network`, the prompt explicitly warns about real network.

## Validation
- `cargo test cli::decide` with injected ttyness + scripted input (a trait over the input
  source) covering every branch.

## Dependencies
05, 22, 23. (Reads `execute_after`/ttyness intent from the policy (05); renders via 22/23.)

## Non-goals
No drift check or real execution (25).

## Design References
DESIGN §4.4 (gate), §8 (state machine), §10.2 T8, ADR-003/005.
