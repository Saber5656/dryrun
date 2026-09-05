# 04 — Config file loading and precedence

## Title
Config file loading and precedence

## Summary
Discover, parse, and validate the TOML config file, and implement the
defaults < config < env < flags precedence (DESIGN §5.2, §12), including the security rule
that project-level config may only *narrow* dangerous scope, never widen it.

## Context
Config is an untrusted input when it comes from a cloned repo (DESIGN §10.5 A3). It must be
schema-validated, unable to execute code, and unable to silently escalate the writable
scope. This issue owns that boundary (DESIGN §10.3 config parser).

## Scope
- In: `src/config/mod.rs` (discovery + merge), `src/config/schema.rs` (`Config` serde
  struct), env var reading (`DRYRUN_*`), and a `RawSettings` merge result consumed by 05.
- Out: turning settings into an `EffectivePolicy` (that's 05), path canonicalization (05).

## Detailed Requirements
1. `Config` struct (serde, `#[serde(deny_unknown_fields)]`) mirroring DESIGN §12 example:
   `network`, `timeout`, `max_output`, `max_disk`, `max_files`, `writable`, `redact`,
   `pager`. All fields `Option<_>` (absent = unset). Reuse the size/duration parsers from
   03.
2. Discovery (unless `--no-config`): if `--config <F>` given, load exactly that (missing =
   exit 40). Else in order: `$DRYRUN_CONFIG`, `./.dryrun.toml` (project), then
   `${XDG_CONFIG_HOME:-$HOME/.config}/dryrun/config.toml` (user). Record each source path
   for `-v` reporting.
3. Parse: read file (cap size at, e.g., 64 KiB; larger = exit 40), `toml::from_str`.
   Any parse error → `DryrunError::Config` with file + line context. Never panic.
4. Env layer: `DRYRUN_NETWORK`, `DRYRUN_TIMEOUT`, `DRYRUN_MAX_OUTPUT`, `DRYRUN_MAX_DISK`,
   `DRYRUN_MAX_FILES`, `DRYRUN_REDACT`, `DRYRUN_PAGER`, `DRYRUN_CONFIG`. Parsed with the
   same parsers; bad value = exit 40.
5. Merge precedence (produce `RawSettings`): built-in defaults (DESIGN §4.2) < user config
   < project config < env < flags. **Security constraint**: project config (`./.dryrun.toml`)
   may set `network = "deny"`, lower limits, and *remove* writable roots, but may NOT add
   writable roots outside the workspace and may NOT set `network = "allow"`; such fields in
   a project config are ignored with a `-v` warning (never an escalation). User config and
   flags may widen (that is the user's explicit machine-level choice).
6. Relative `writable` paths in a config resolve relative to that config file's directory;
   record them as unresolved (05 canonicalizes + containment-checks).
7. Provide `load(args: &ParsedArgs) -> Result<RawSettings, DryrunError>` and a
   `sources: Vec<(Layer, PathBuf)>` for reporting.

## Acceptance Criteria
- A valid `.dryrun.toml` is discovered and merged; unknown key → exit 40.
- `--no-config` ignores all files; `--config missing.toml` → exit 40.
- Precedence: a value set in both user config and flags resolves to the flag; env beats
  config; project beats user (except the narrowing rule).
- Project config with `network = "allow"` or a `writable` outside the workspace is
  ignored (asserted by test) and does not escalate scope.
- Oversized/malformed config → exit 40, no panic.

## Validation
- `cargo test config::` with fixtures: valid, unknown-field, oversized, project-escalation-
  ignored, precedence.
- Manual: run with `-vv` and confirm the effective sources are listed.

## Dependencies
01, 02.

## Non-goals
No canonicalization or dangerous-root enforcement (05). No policy object here.

## Design References
DESIGN §5.2 (precedence), §12 (config), §10.3 (parser boundary), §10.5 A3 (untrusted
config), ADR-004 (scope).
