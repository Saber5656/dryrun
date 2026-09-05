# 13 — Landlock ruleset builder

## Title
Landlock ruleset builder (Linux)

## Summary
Implement the shared Landlock ruleset that confines the rehearsed process's filesystem
*writes* to the writable roots + private tmp while allowing broad reads, used by both Linux
engines as the kernel-enforced half of workspace confinement (DESIGN §10.2 T1/T4,
ADR-004).

## Context
overlayfs only captures writes under its mount; writes elsewhere would hit real files, so
Landlock is a hard requirement for the primary engine and the confinement mechanism for the
portable engine (research/01). ABI varies by kernel; probe at runtime (DESIGN §2.4 U1).

## Scope
- In: `src/engine/linux/landlock.rs` — `build_ruleset(policy, resolved_write_roots) ->
  LandlockRules` and `apply(rules)` (called in the child after fork, before exec).
- Out: seccomp (18), namespaces/overlay (20/21), macOS.

## Detailed Requirements
1. Use the `landlock` crate. **Landlock semantics**: only the access rights placed in the
   ruleset's *handled* set are restricted; any right NOT handled is left completely
   unrestricted by Landlock. To confine writes while leaving reads unrestricted in v1, the
   handled set MUST contain only write-family rights and MUST NOT contain
   `LANDLOCK_ACCESS_FS_READ_FILE`/`READ_DIR`/`EXECUTE`. Concretely:
   - **Handled (restricted) rights**: `WRITE_FILE`, `MAKE_REG`, `MAKE_DIR`, `MAKE_SYM`,
     `MAKE_CHAR`, `MAKE_BLOCK`, `MAKE_FIFO`, `MAKE_SOCK`, `REMOVE_FILE`, `REMOVE_DIR`, plus
     (ABI ≥ v2) `REFER` and (ABI ≥ v3) `TRUNCATE`. Do **not** handle read/exec rights.
   - **Allow rules**: grant the full handled write set ONLY on each writable root and the
     private tmp. No other path gets write rights → writes elsewhere are denied; reads
     everywhere remain unaffected because read rights are not handled.
   - Use the crate's `BestEffort` compatibility so that on older ABIs the unavailable
     rights (REFER/TRUNCATE) are simply not enforced rather than erroring — but never fall
     back to handling *fewer* write rights than "deny writes outside roots"; if the base
     write rights cannot be handled at all, the builder reports unusable (fail-closed).
2. `apply` is called in the child post-`fork`, after `no_new_privs` is set, before
   `execvp`. On any Landlock error, the child must fail-closed (exit with a code the
   parent maps to `SANDBOX_INIT_FAILED` 42) — never exec unconfined.
3. Best-effort enforcement policy: if the running kernel's Landlock ABI is older than what
   a rule needs, degrade *safely* (still deny writes outside roots) and record a caveat via
   the engine, rather than silently granting more than intended. If Landlock is entirely
   absent, this builder reports unusable and the engine's `probe` fails (portable engine
   then cannot run → selection errors per 07).
4. Provide `resolve_write_roots(policy, layout)` translating the policy's writable roots
   into the concrete paths the child sees (shadow paths for portable; overlay merged path
   for overlay — the caller passes the right set).
5. Expose which ABI/level was applied so 27 (doctor) and reports can show it.
6. All syscalls via the crate; any raw `unsafe` gets a `// SAFETY:` comment.

## Acceptance Criteria
- With the ruleset applied, a child writing under a writable root succeeds; a child writing
  to `/tmp/outside` or `$HOME/x` gets EACCES (test on a Landlock-capable kernel).
- Reads outside the roots succeed (broad read allowance).
- Landlock init failure in the child causes fail-closed (parent sees 42), never an
  unconfined exec.
- The applied ABI level is reported.

## Validation
- `cargo test --test landlock_it` (os+kernel gated; skipped with a clear message on
  unsupported kernels) asserting write-allowed-inside / write-denied-outside / read-allowed.
- CI matrix (33) runs this on a Landlock-capable image.

## Dependencies
05, 06.

## Non-goals
No network control (18/20); no read confinement (v2); no macOS.

## Design References
DESIGN §10.2 T1/T4, §10.7, §3.5, §2.4 U1, ADR-004, research/01 (Landlock ABI).
