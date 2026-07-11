# 05 — Policy model and path containment

## Title
Policy model and path containment

## Summary
Build the `EffectivePolicy` (DESIGN §5) from parsed args + raw settings, and implement the
security-critical path canonicalization / containment / dangerous-root rules (DESIGN §5.1)
that every engine relies on as defense-in-depth.

## Context
`EffectivePolicy` is the single object all engines consume (DESIGN §5). The path
containment logic here is the userspace half of the write-confinement guarantee; the
kernel (Landlock/SBPL) is the enforcing half (DESIGN §10.2 T1/T4). Both must agree.

## Scope
- In: `src/policy/mod.rs` (`EffectivePolicy`, builder), `src/policy/limits.rs`
  (`ResourceLimits`), `src/policy/path.rs` (canonicalize, contains, dangerous-root deny).
- Out: engine enforcement (13/16/17/19/20/21), monitor (10).

## Detailed Requirements
1. Types exactly as DESIGN §5: `EffectivePolicy`, `NetworkPolicy{Deny,Allow}`,
   `ResourceLimits`, `CommandSpec`, `ExecuteAfter{Ask,Yes,Never}`,
   `ViolationMode{Report,Fail}`.
2. `build(args: &ParsedArgs, settings: &RawSettings) -> Result<EffectivePolicy,
   DryrunError>`:
   - `workspace` = `--workdir` or cwd, then `canonicalize`. If it does not exist and it is
     the cwd/workdir, error 40 (we don't create the workspace root).
   - `writable_roots` = `[workspace]` + each `--writable`/config writable, each
     canonicalized. Always append the private tmp dir path (created by the engine at
     prepare; here just reserve/represent it).
   - `network` from settings (default Deny). `--allow-network` → Allow.
   - `limits` from settings/flags with DESIGN §4.2 defaults.
   - `execute_after`: `--yes`→Yes, `--no-execute`→Never, else Ask.
   - `violation_mode`: default Report; the `--fail-on-violation` flag (owned by issue 03) →
     Fail. This issue only consumes the parsed flag; it does not define new flags.
3. `path.rs`:
   - `canonicalize_existing(p)` and `canonicalize_lexical(p)` (for not-yet-existing paths:
     normalize `.`/`..` without touching FS, then verify no `..` escapes a known root).
   - `contains(root, path) -> bool`: true iff canonical `path` is `root` or a descendant.
     Must resolve symlinks and reject `..` traversal and symlink-into-outside.
   - `DANGEROUS_ROOTS`: `/`, `$HOME` (the home root itself), `/usr`, `/etc`, `/bin`,
     `/sbin`, `/lib`, `/System`, `/Library`, `/opt/homebrew`, `/nix`, `/boot`, `/dev`,
     `/proc`, `/sys`. A `--writable` equal to (or an ancestor of) a dangerous root, or
     equal to `$HOME`, is rejected (exit 40) unless `--writable-force` is set (add this
     flag; document as discouraged in 03/37). The workspace itself is exempt from the
     "ancestor of dangerous root" check only if it is a normal project dir (i.e. reject
     `--workdir /`).
4. Determinism: `writable_roots` deduplicated and sorted; nested roots collapsed (a child
   of another writable root is redundant — keep the ancestor, drop the child, note it).
5. All functions return typed errors; no panics on bad input.

## Acceptance Criteria
- Building a policy in a normal project dir yields workspace=cwd, writable=[cwd,+tmp],
  network=Deny by default.
- `--writable /` and `--writable $HOME` → exit 40; with `--writable-force` they are
  allowed but flagged (a field records the override for the report/renderer).
- `contains()` rejects `workspace/../secret`, a symlink inside workspace pointing to
  `/etc`, and `/etc/passwd`; accepts `workspace/sub/file`.
- Nested/duplicate writable roots are collapsed deterministically.
- `--workdir /` → exit 40.

## Validation
- `cargo test policy::` including a `path::contains` table with symlink and `..` cases
  (create temp trees).
- Property test (proptest, optional) that `contains(root, join(root, rel))` holds for
  non-escaping `rel` and fails for escaping ones.

## Dependencies
01, 02, 03, 04. (Consumes 03's `ParsedArgs` and 04's `RawSettings`, so both must exist to
compile `build(args, settings)`.)

## Non-goals
No kernel enforcement; no engine calls; no filesystem mutation beyond canonicalize.

## Design References
DESIGN §5 (policy), §5.1 (containment), §10.2 T1/T4, §10.5 A2 (dangerous writable),
ADR-004.
