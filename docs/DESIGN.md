# dryrun — Design

Version: v1 design, 2026-07-10
Status: authoritative source of truth for implementation
Audience: implementation agents (assume low context; be literal)

> One-line concept: **rehearse a command in an OS-native sandbox and show the filesystem
> changes as a diff before you run it for real.**

This document is the canonical specification. `docs/ISSUE_PLAN.md` slices it into issues;
`docs/issues/*.md` are the executable drafts. If anything here is ambiguous, prefer the
most conservative (fail-closed, least-privilege) reading and record the decision in an
ADR.

---

## 1. Product overview

### 1.1 Problem

Destructive or wide-reaching shell commands (`rm -rf`, `find … -delete`, mass
`sed -i`, build scripts, `git clean -fdx`, database migrations, installers) are run blind.
The user cannot see what will change until after it has changed. Undo is often impossible.

### 1.2 Solution

`dryrun <command>`:

1. Runs `<command>` inside an OS-native sandbox where **writes are captured** into a
   throwaway shadow of the workspace and **network is blocked by default**.
2. Computes a **ChangeSet** (created / modified / deleted / renamed files, mode changes,
   and blocked out-of-workspace / network attempts).
3. Renders a human diff (and/or a machine-readable JSON report).
4. On approval, **re-executes the same command for real** (ADR-005). Preview-only mode is
   available.

### 1.3 Primary users (both, human-interactive prioritized)

- **Human developer at a terminal**: the headline experience — a clear, skimmable diff of
  what a scary command would do, then approve/deny. Prioritized for v1 polish.
- **AI agent / automation as a guard step**: proposes a command, dryrun rehearses it,
  emits JSON, a human (or policy) approves. v1 ships a stable non-interactive JSON
  contract and meaningful exit codes so this works, but interactive polish comes first.

### 1.4 Design pillars

1. **Safe by default**: network denied, writes confined to the workspace, fail-closed if
   the sandbox cannot be established. (ADR-003, ADR-004)
2. **Honest about fidelity**: the diff is a *prediction*. The tool names the engine used,
   states guarantees per engine, and flags divergence risk. (ADR-001, ADR-005)
3. **No external runtime**: OS-native primitives only; a single binary. (ADR-001)
4. **Reviewable**: every out-of-workspace or network attempt is surfaced, not hidden.

---

## 2. Scope

### 2.1 v1 goals (in scope)

- G1. Rehearse a single command line (with a shell, `sh -c "<string>"` semantics, and
  argv form) on macOS and Linux.
- G2. Three sandbox engines behind one trait: `macos-seatbelt`, `linux-overlay`,
  `linux-portable`, with automatic selection + explicit override.
- G3. Workspace-scoped write capture; out-of-workspace writes **denied** (kernel-enforced)
  and reported **best-effort** with an explicit fidelity caveat (see §10.3 reporting note).
- G4. Network default-deny with `--allow-network`.
- G5. ChangeSet computation with created/modified/deleted/renamed/mode-change + content
  diffs for text, summaries for binary.
- G6. Human diff renderer (colored, paged, `NO_COLOR`/TTY aware) and machine JSON report
  (versioned schema).
- G7. Approve-then-reexecute flow with `--yes`, `--no-execute`, and a pre-real-run drift
  check.
- G8. `dryrun doctor` capability probe + remediation guidance (esp. Ubuntu 24.04 userns).
- G9. Config file + flags with a documented precedence order.
- G10. Security model implemented and tested: fail-closed engine init, escape-resistant
  policy, resource limits (output size, disk, wall-clock, file count).
- G11. OSS release baseline: tests, CI (macOS + Linux), `cargo-deny`, `SECURITY.md`,
  `CONTRIBUTING.md`, license, man page + completions, reproducible `--locked` release
  binaries via GitHub Releases.

### 2.2 v1 non-goals (explicitly out)

- N1. Windows support.
- N2. Read confinement / secret-read blocking **and** read *reporting* (reads are broad in
  v1; the v1 primitives provide no reliable read-observation channel, so v1 neither blocks
  nor reports reads — both are v2). Secret *contents* are still redacted from reports
  (§10.4).
- N3. Promoting captured changes without re-execution (ADR-005).
- N4. Rehearsing interactive TUIs / long-lived servers as their "final state" (the tool
  runs them, captures until exit/timeout; it does not model a server's steady state).
- N5. Full-system (outside-workspace) write capture.
- N6. Rich TUI, multi-command scripts as first-class plans, distributed/remote execution.
- N7. GUI, editor plugins, language-specific integrations.
- N8. Undo/rollback of the real run.

### 2.3 v2 deferred ideas (design must not preclude)

- V2-1. Read confinement + secret-read denial **and read reporting** (Landlock read rules /
  SBPL read denies + a read-observation channel; v1 tracks no reads).
- V2-2. Change promotion for file-only commands with conflict detection.
- V2-3. Interactive network-allow prompt on first blocked socket.
- V2-4. `ratatui` TUI review surface.
- V2-5. Additional engines (FreeBSD `jail`/Capsicum; a Seatbelt replacement if Apple ships
  one).
- V2-6. Rehearse a sequence/script as a plan with per-step diffs.
- V2-7. Policy profiles (named allowlists) and org-shared policy files.
- V2-8. Content-addressed cache of rehearsals.

### 2.4 Known unknowns (may spawn issues during implementation)

- U1. Exact Landlock ABI available on target kernels → runtime probe, not version parse.
  May require per-ABI code paths.
- U2. macOS Tahoe+ SBPL behavior drift (new sysctls, path changes) → golden tests + an
  allow-default profile reduce but do not remove risk.
- U3. Seatbelt long-term availability (deprecated) → engine trait isolates blast radius.
- U4. Overlayfs whiteout/opaque encoding under `userxattr` across kernels → verify with
  integration tests on the CI kernel matrix; may need per-kernel handling.
- U5. Shells reading unexpected sysctls/paths breaking even allow-default profiles.
- U6. `clonefile` behavior on non-APFS/ExFAT/network mounts → copy fallback + tests.
- U7. Detecting "failed due to blocked network" reliably across tools → heuristic; may
  need a curated signature list that grows.
- U8. Symlink/hardlink edge cases in changeset reconstruction on copy engines.

---

## 3. Architecture

### 3.1 Process model

```
dryrun (parent process)
 ├─ parse CLI + config  → EffectivePolicy
 ├─ select SandboxEngine (probe / override)
 ├─ engine.prepare()    → RunContext (shadow/overlay dirs, run dir)
 ├─ fork/spawn child:
 │     child: enter sandbox (namespaces / seatbelt / landlock / seccomp)
 │            chdir(workspace-or-shadow); exec command; stdio piped
 │     parent: tee child stdout/stderr → capture buffers (bounded)
 │             monitor: wall-clock timeout, disk/inode budget
 ├─ wait(child) → ExecOutcome (exit status, signals, limits hit)
 ├─ engine.scan_changes() → ChangeSet
 ├─ render (human diff | JSON report)
 ├─ decision gate (approve / deny / --yes / --no-execute / JSON caller)
 ├─ if approved: drift check → real exec (no sandbox) → stream
 └─ engine.cleanup() (always, even on error/panic path)
```

### 3.2 Module / file layout (crate: `dryrun`)

Single binary crate with an internal library (`src/lib.rs`) so tests can exercise
internals. Suggested layout — implementation issues map 1:1 to these files:

```
Cargo.toml                      # metadata, deps, lints, release profile
rust-toolchain.toml
deny.toml                       # cargo-deny config
src/
  main.rs                       # thin: parse args, call cli::run, map error→exit code
  lib.rs                        # module wiring, pub use
  cli/
    mod.rs                      # clap command/flag definitions
    args.rs                     # parsed args → typed struct
    run.rs                      # top-level orchestration (the pipeline in 3.1)
  config/
    mod.rs                      # config file discovery, parse, merge precedence
    schema.rs                   # Config struct (serde, deny_unknown_fields)
  policy/
    mod.rs                      # EffectivePolicy: writable roots, network, limits
    limits.rs                   # ResourceLimits (output/disk/inodes/walltime)
    path.rs                     # path normalization/containment (canonicalize, no ..)
  engine/
    mod.rs                      # SandboxEngine trait, EngineKind, RunContext, ExecOutcome
    select.rs                   # capability probe + selection precedence
    common/
      shadow.rs                 # copy/reflink shadow builder (macos + portable)
      scan.rs                   # tree-diff changeset scanner (copy engines)
      stdio.rs                  # bounded tee capture
      monitor.rs                # walltime + disk/inode budget monitor
    macos/
      mod.rs                    # macos-seatbelt engine
      seatbelt.rs               # SBPL profile builder
      ffi.rs                    # sandbox_init/clonefile extern "C" (unsafe, SAFETY docs)
    linux/
      mod.rs                    # engine dispatch (overlay vs portable)
      overlay.rs                # linux-overlay engine (userns+mount+overlayfs+netns)
      portable.rs               # linux-portable engine (shadow+landlock+seccomp)
      landlock.rs               # Landlock ruleset builder (shared)
      seccomp.rs                # socket-family deny filter (portable)
      namespace.rs              # unshare/mount helpers (overlay)
  changes/
    mod.rs                      # ChangeSet, FileChange, ChangeKind types
    diff.rs                     # text/binary diff computation
    classify.rs                # text-vs-binary, size caps, redaction hooks
  report/
    mod.rs                      # Report struct + versioned JSON schema (serde)
    human.rs                    # colored/paged human renderer
    json.rs                     # JSON emitter (schema_version pinned)
  execreal/
    mod.rs                      # post-approval real execution + drift check
  doctor/
    mod.rs                      # `dryrun doctor` implementation
  error.rs                      # DryrunError (thiserror), exit-code mapping
  version.rs                    # version/build metadata
docs/                           # this design set (source of truth)
tests/                          # integration tests (per engine, gated by cfg/os)
```

### 3.3 Engine abstraction

```rust
/// A sandbox engine rehearses a command and reports the changes it would make.
/// All methods must be fail-closed: if isolation cannot be guaranteed, return Err.
pub trait SandboxEngine {
    /// Stable identifier used in reports and logs.
    fn kind(&self) -> EngineKind;

    /// Cheap check: can this engine run on this machine right now?
    /// Must not mutate system state. Used by the selector and by `doctor`.
    fn probe(&self) -> Probe;

    /// Build the isolated environment (shadow/overlay dirs, run dir).
    /// Returns a RunContext the caller passes to `execute`.
    fn prepare(&self, policy: &EffectivePolicy) -> Result<RunContext, DryrunError>;

    /// Spawn the command inside the sandbox and wait for it. Streams stdio through
    /// the provided sinks (bounded). Enforces limits via the monitor.
    fn execute(&self, ctx: &RunContext, cmd: &Command, io: &mut StdioSinks)
        -> Result<ExecOutcome, DryrunError>;

    /// Compute the ChangeSet from the finished sandbox state.
    fn scan_changes(&self, ctx: &RunContext, policy: &EffectivePolicy)
        -> Result<ChangeSet, DryrunError>;

    /// Tear down all resources. MUST be idempotent and safe to call after any prior
    /// step failed. MUST NOT touch anything outside the run dir / shadow / overlay.
    fn cleanup(&self, ctx: RunContext) -> Result<(), DryrunError>;
}

pub enum EngineKind { MacosSeatbelt, LinuxOverlay, LinuxPortable }

pub struct Probe {
    pub usable: bool,
    pub reasons: Vec<String>,      // human-readable blockers if !usable
    pub degraded: Vec<String>,     // works but with caveats (e.g. copy fallback)
}
```

`RunContext`, `ExecOutcome`, `Command`, `StdioSinks` are defined in `engine/mod.rs`;
their fields are specified in the relevant issues.

### 3.4 Engine selection

`engine::select::choose(policy) -> (Box<dyn SandboxEngine>, SelectionReport)`:

Precedence:

1. If `--engine <kind>` given: use it; if its `probe().usable == false`, **error**
   (never silently downgrade an explicit choice). Exit code = `CONFIG_ERROR`.
2. Else on macOS: `macos-seatbelt` (degraded→copy if non-APFS). If Seatbelt cannot init,
   **error** (fail-closed); do not run unsandboxed.
3. Else on Linux: try `linux-overlay`; if `probe().usable == false` (e.g. userns blocked),
   fall back to `linux-portable`. If neither usable, **error** with remediation text.
4. On any other OS: **error** `UNSUPPORTED_PLATFORM`.

The `SelectionReport` (chosen kind, why, degraded caveats, rejected engines + reasons) is
included in the report header and shown by `doctor`.

### 3.5 Per-engine guarantee matrix

| Capability | macos-seatbelt | linux-overlay | linux-portable |
|---|---|---|---|
| Capture created/modified files in workspace | yes | yes (upperdir) | yes (post-scan) |
| Capture deletions | yes (tree diff) | yes (whiteouts) | yes (tree diff) |
| Capture renames as rename (not add+del) | best-effort (inode/hash heuristic) | best-effort | best-effort |
| Mode/permission changes | yes | yes | yes |
| Symlink create/change | yes | yes | yes |
| Deny out-of-workspace writes (kernel-enforced) | yes (SBPL) | yes (Landlock) | yes (Landlock) |
| Report each blocked write (best-effort) | best-effort + caveat | best-effort + caveat | best-effort + caveat |
| Network default-deny (external IP) | yes (SBPL) | yes (netns, lo up) | yes (seccomp AF_INET/INET6/PACKET) |
| Local IPC (AF_UNIX/AF_NETLINK) allowed | yes | yes | yes (residual host-daemon caveat) |
| Loopback TCP during rehearsal | yes (localhost allow) | yes (lo up) | no (AF_INET blocked; unix-socket IPC only) |
| Runs at original absolute workspace path | no (shadow path) | yes (overlay at path) | no (shadow path) |
| Requires unprivileged userns | no | yes | no |
| Min OS | macOS 13 + APFS (else copy) | kernel ≥ 5.13 + userns allowed | kernel ≥ 5.13 |

"best-effort rename" and the absolute-path caveat are surfaced in every report so users
never over-trust a prediction.

---

## 4. CLI specification

### 4.1 Synopsis

```
dryrun [GLOBAL OPTS] <command> [args...]
dryrun [GLOBAL OPTS] -- <command line as one shell string>
dryrun doctor [--json]
dryrun version [--json]
dryrun completions <shell>
```

Command capture:

- `dryrun rm -rf build` → argv form: exec `rm` with `["-rf","build"]`, no shell.
- `dryrun -c "rm -rf build && make"` → shell form: `sh -c "<string>"`.
- Everything after the first non-option token (or after `--`) belongs to the command, not
  to dryrun. Document this precedence explicitly; test it.

### 4.2 Global options

| Flag | Type | Default | Meaning |
|---|---|---|---|
| `-c, --command <STR>` | string | — | Run `<STR>` via `sh -c`. Mutually exclusive with argv form |
| `-C, --workdir <DIR>` | path | cwd | Workspace root (writable + default cwd) |
| `--writable <DIR>` | path (repeatable) | — | Add a writable/captured root |
| `--writable-force` | bool | false | Allow a dangerous `--writable` root (e.g. `$HOME`); discouraged, echoed in report |
| `--allow-network` | bool | false | Permit full outbound network during rehearsal (ADR-003) |
| `--fail-on-violation` | bool | false | Exit 30 if any out-of-workspace write / blocked network attempt is observed |
| `--no-drift-check` | bool | false | Skip the pre-real-run drift check (discouraged; recorded in output) |
| `--keep-run-dir` | bool | false | Do not delete the run dir on teardown (debugging) |
| `--engine <KIND>` | enum | auto | Force `macos-seatbelt`\|`linux-overlay`\|`linux-portable` |
| `--yes, -y` | bool | false | Auto-approve the real run after rehearsal |
| `--no-execute` | bool | false | Preview only; never run for real |
| `--json` | bool | false | Emit JSON report to stdout; implies non-interactive |
| `--output <FILE>` | path | — | Write JSON report to FILE (with or without `--json`) |
| `--no-color` | bool | env | Disable ANSI color (also honors `NO_COLOR`, non-TTY) |
| `--no-pager` | bool | false | Do not page human output |
| `--max-output <BYTES>` | size | 10MiB | Cap captured stdout+stderr each |
| `--max-disk <BYTES>` | size | 1GiB | Abort rehearsal if shadow/upper exceeds |
| `--max-files <N>` | int | 100000 | Abort if changeset exceeds N files |
| `--timeout <DUR>` | duration | 120s | Wall-clock limit for the rehearsal |
| `--config <FILE>` | path | discovery | Explicit config file |
| `--no-config` | bool | false | Ignore all config files |
| `-v, --verbose` | count | 0 | Increase log verbosity (stderr) |
| `-q, --quiet` | bool | false | Suppress non-essential stderr |

Sizes accept `10MiB`, `1GB`, etc.; durations accept `120s`, `2m`.

### 4.3 Exit codes

| Code | Name | Meaning |
|---|---|---|
| 0 | OK | Rehearsal (and real run, if executed) completed; user approved or `--no-execute` |
| 10 | REHEARSAL_COMMAND_FAILED | The rehearsed command exited non-zero (rehearsal itself worked) |
| 20 | DENIED_BY_USER | User rejected the change at the decision gate |
| 30 | POLICY_VIOLATION | Command attempted out-of-workspace write / network while blocked, and policy is set to fail on violation |
| 40 | CONFIG_ERROR | Bad flags/config, mutually exclusive options, forced engine unusable |
| 41 | UNSUPPORTED_PLATFORM | OS/engine unavailable and no fallback |
| 42 | SANDBOX_INIT_FAILED | Fail-closed: sandbox could not be established |
| 43 | LIMIT_EXCEEDED | timeout/disk/inode/output cap hit during rehearsal |
| 50 | REAL_RUN_FAILED | Approved real execution exited non-zero |
| 70 | INTERNAL_ERROR | Unexpected internal error (bug) |

Exit codes are a stable contract (agents depend on them). Changing them is a breaking
change. When `--json`, the same code is mirrored in the report's `exit.code` field.

### 4.4 Interactive decision gate

When stdout is a TTY and neither `--yes`, `--no-execute`, nor `--json` is set, after
rendering the diff dryrun prompts:

```
Apply this for real? [y]es / [N]o / [d]etails / [q]uit:
```

- `y` → drift check → real run.
- `N`/`q`/EOF → exit `DENIED_BY_USER` (20).
- `d` → re-render full diff (e.g. re-open pager) then re-prompt.

Non-TTY without `--yes`/`--no-execute` defaults to **no real run** (safe default) and exits
`OK` with a note that no approval channel was available (or `DENIED_BY_USER` if a policy
requires an explicit decision — decided in the CLI issue, default: safe/no-run).

---

## 5. Policy model

`EffectivePolicy` is the merged, validated result of config + flags. It is the single
object every engine consumes.

```rust
pub struct EffectivePolicy {
    pub workspace: PathBuf,             // canonical
    pub writable_roots: Vec<PathBuf>,   // canonical, includes workspace + private tmp
    pub network: NetworkPolicy,         // Deny | Allow
    pub limits: ResourceLimits,
    pub command: CommandSpec,           // Argv{prog,args} | Shell{string}
    pub engine_override: Option<EngineKind>,
    pub execute_after: ExecuteAfter,    // Ask | Yes | Never
    pub violation_mode: ViolationMode,  // Report | Fail  (drives exit 30)
}

pub struct ResourceLimits {
    pub max_output_bytes: u64,   // per stream
    pub max_disk_bytes: u64,
    pub max_files: u64,
    pub wall_timeout: Duration,
}
```

### 5.1 Path containment rules (security-critical)

- All workspace/writable roots are `canonicalize`d at policy build; a path that fails to
  canonicalize (missing) is created if it is the workspace, else rejected.
- Writable roots must not include `/`, the user home root, or system roots (`/usr`,
  `/etc`, `/bin`, `/System`, `/nix`, `/opt/homebrew`, `%`-prefixed on none)…: a **deny
  list of dangerous roots** rejects obviously unsafe `--writable` values unless
  `--writable-force` (documented, discouraged). Rationale: prevent a user from trivially
  turning off the core protection by accident.
- Containment checks operate on canonical paths and reject any traversal
  (`..`, symlink-into-outside). Engines re-enforce at the kernel level (Landlock/SBPL) so
  a bug in the userspace check cannot defeat isolation (defense in depth).

### 5.1.1 Temp-directory handling (ADR-004)

The `writable_roots` always include a private, discarded temp dir. `$TMPDIR` is remapped to
it so `$TMPDIR`-based writes work and are discarded. On the **overlay** engine, `/tmp` is
additionally remapped transparently (private tmpfs mount). On the **copy** engines
(macOS, portable), a hard-coded absolute write to real `/tmp/...` is **denied** (and
reported best-effort, per the §10.3 fidelity note) like any other out-of-workspace write
(user decision 2026-07-11) — it does not silently reach the real `/tmp`. A user who wants
real `/tmp` captured passes `--writable /tmp`.

### 5.2 Precedence

`built-in defaults < config file < environment (DRYRUN_*) < command-line flags`.
The doctor/report shows the effective values and where each came from when `-v`.

---

## 6. ChangeSet data model

```rust
pub struct ChangeSet {
    pub engine: EngineKind,
    pub root: PathBuf,                    // workspace
    pub files: Vec<FileChange>,          // sorted, deterministic order
    pub blocked_writes: Vec<BlockedWrite>,   // out-of-workspace attempts
    pub network_attempts: Vec<NetworkAttempt>,
    pub truncated: bool,                 // hit max_files
    pub caveats: Vec<String>,            // engine-specific fidelity notes
}

pub struct FileChange {
    pub path: PathBuf,                   // relative to root
    pub kind: ChangeKind,
    pub old_meta: Option<Meta>,
    pub new_meta: Option<Meta>,
    pub diff: Option<Diff>,              // for text modifications
}

pub enum ChangeKind {
    Created, Modified, Deleted,
    Renamed { from: PathBuf },
    ModeChanged,                          // perms/owner only, content identical
    TypeChanged,                          // e.g. file→symlink
    SymlinkChanged,
}

pub struct Meta {
    pub size: u64,
    pub mode: u32,
    pub is_symlink: bool,
    pub symlink_target: Option<PathBuf>,
    pub content_hash: Option<[u8; 32]>,  // blake3, None if too large/skipped
}

pub enum Diff {
    Text { unified: String, added: u64, removed: u64 },
    Binary { old_size: u64, new_size: u64 },
    TooLarge { size: u64 },
    Redacted,                            // matched a secret-path/pattern rule
}

pub struct BlockedWrite { pub path: PathBuf, pub op: String } // op: create/write/unlink/…
pub struct NetworkAttempt { pub summary: String }             // e.g. "connect AF_INET :443"
```

### 6.1 Change detection per engine

- **linux-overlay**: enumerate the overlay upperdir. Regular file in upper = created or
  modified (compare against lowerdir by hash to classify + diff). Char device 0:0 =
  deletion (whiteout). `user.overlay.opaque` xattr = directory replaced. This is the
  highest-fidelity path.
- **copy engines (macos/portable)**: walk the pre-run snapshot manifest (path→Meta hash
  captured cheaply at `prepare`) vs post-run tree. Classify by presence + hash. Renames
  inferred when a deleted path's content hash equals a created path's hash (heuristic,
  reported as best-effort).
- **blocked_writes / network_attempts**: sourced from the sandbox's violation channel
  (SBPL report mode / Landlock EACCES observation via a thin exec wrapper that records
  denied syscalls where feasible / seccomp `SECCOMP_RET_ERRNO` with an audit tap). Where a
  kernel primitive cannot enumerate every attempt, the report says so in `caveats` rather
  than implying completeness.

### 6.2 Determinism

Files are sorted by path; diffs use a fixed algorithm; hashes are blake3. The same
rehearsal on the same inputs yields byte-identical JSON (modulo timestamps, which live in
a dedicated `meta` block). This makes golden/snapshot tests possible.

---

## 7. JSON report schema (v1)

Top-level object, `schema_version` is a string and **the compatibility contract**:

```json
{
  "schema_version": "1.0",
  "meta": {
    "tool_version": "1.0.0",
    "os": "macos|linux",
    "engine": "macos-seatbelt|linux-overlay|linux-portable",
    "engine_selection": {
      "chosen": "linux-portable",
      "reason": "user namespaces blocked by AppArmor",
      "degraded": ["copy fallback: workspace not on reflink fs"],
      "rejected": [{"engine":"linux-overlay","reason":"clone_newuser EPERM"}]
    },
    "started_at": "2026-07-10T12:00:00Z",
    "duration_ms": 1234,
    "command": {"form":"argv","program":"rm","args":["-rf","build"]},
    "policy": {
      "workspace": "/abs/path",
      "writable_roots": ["/abs/path"],
      "network": "deny",
      "limits": {"max_output_bytes":10485760,"max_disk_bytes":1073741824,
                 "max_files":100000,"wall_timeout_ms":120000}
    }
  },
  "exit": {"phase":"rehearsal","code":10,"name":"REHEARSAL_COMMAND_FAILED",
           "child_exit":1,"signal":null,"limit_hit":null},
  "changes": {
    "truncated": false,
    "caveats": ["renames are best-effort","command ran at a shadow path"],
    "files": [
      {"path":"build/app","kind":"deleted","old":{"size":123,"mode":33188},
       "new":null,"diff":null}
    ],
    "blocked_writes": [{"path":"/Users/x/.zshrc","op":"write"}],
    "network_attempts": [{"summary":"connect AF_INET 93.184.216.34:443"}]
  },
  "summary": {"created":0,"modified":0,"deleted":1,"renamed":0,
              "mode_changed":0,"blocked_writes":1,"network_attempts":1}
}
```

Rules:

- Additive changes bump the minor version (`1.1`); breaking changes bump major (`2.0`) and
  are avoided within v1. Consumers must ignore unknown fields.
- `--json` writes exactly this object to stdout and nothing else (logs go to stderr).
- The schema is captured as a committed JSON Schema file (`docs/report.schema.json`) and a
  snapshot test keeps it in sync.

---

## 8. Run state machine

States and transitions (the orchestrator in `cli/run.rs`):

```
INIT
  → PARSED            (args+config valid)            | → FAILED(CONFIG_ERROR)
PARSED
  → ENGINE_SELECTED   (engine chosen+usable)         | → FAILED(UNSUPPORTED_PLATFORM|CONFIG_ERROR)
ENGINE_SELECTED
  → PREPARED          (prepare ok)                   | → FAILED(SANDBOX_INIT_FAILED)
PREPARED
  → REHEARSED         (child ran to exit)            | → FAILED(SANDBOX_INIT_FAILED|LIMIT_EXCEEDED)
REHEARSED
  → SCANNED           (changeset built)              | → FAILED(INTERNAL_ERROR|LIMIT_EXCEEDED)
SCANNED
  → RENDERED          (human/json emitted)
RENDERED
  → decision:
        --no-execute / non-TTY-no-approval → DONE(OK)
        JSON caller / policy               → DECIDED_EXTERNAL
        denied                             → DONE(DENIED_BY_USER)
        approved / --yes                   → DRIFT_CHECK
DRIFT_CHECK
  → REAL_RUNNING      (no blocking drift, or user re-confirms)
  → DONE(DENIED_BY_USER) (user aborts on drift warning)
REAL_RUNNING
  → DONE(OK)          (real exit 0)
  → DONE(REAL_RUN_FAILED) (real exit non-0)
ANY
  → always runs cleanup() on the way to DONE/FAILED (even on panic: teardown guard)
```

Invariants:

- `cleanup()` runs exactly once for a successful `prepare()`, via an RAII guard, even on
  early return, error, signal (SIGINT/SIGTERM handled: teardown then exit), or panic
  (teardown in a `Drop`; `panic = "abort"` decision revisited so teardown is not skipped —
  see release issue).
- No real execution can occur before `RENDERED` + explicit approval.
- `POLICY_VIOLATION` (30) is emitted from `SCANNED` when `violation_mode == Fail` and
  blocked writes / network attempts exist.

---

## 9. Diff rendering (human)

- Header: engine used, selection caveats, command, workspace, network mode.
- Summary line: `+C ~M -D  ⧉R  ⚠B blocked  🌐N network` counts.
- Per file: `A`/`M`/`D`/`R`/`chmod` badge, path, then for text modifications a unified
  diff (context 3), color via `anstyle` respecting `NO_COLOR`/TTY/`--no-color`.
- Binary/too-large: size delta only.
- Blocked writes and network attempts get their own clearly-marked sections (these are the
  "pay attention" signals).
- Paging: pipe through `$PAGER` (default `less -R`) when TTY and output is long, unless
  `--no-pager`.
- Never render raw file contents that matched a redaction rule; show `‹redacted›`.

---

## 10. Security model

Security is a v1 requirement, not a later pass. This section is normative; issues carry
matching acceptance criteria.

### 10.1 Assets to protect

1. The user's real filesystem outside the workspace (integrity).
2. The user's data that must not leak over the network during a *rehearsal* (ADR-003).
3. The user's secrets (SSH/cloud creds) — not read-blocked in v1, but never written into
   reports in cleartext (redaction) and never transmitted.
4. Integrity of the tool's own artifacts (release binaries) against supply-chain tampering.

### 10.2 Threat model (rehearsal runs untrusted/dangerous commands)

The rehearsed command is treated as **adversarial**: it may try to escape the sandbox,
write outside the workspace, reach the network, exhaust resources, or trick the user via
the diff. Trust boundaries:

```
[ user + dryrun parent ]  ── trusted
        │  (policy, engine setup)
        ▼
[ sandbox boundary: seatbelt / namespaces+landlock+seccomp ]  ── enforcement
        │
        ▼
[ rehearsed command ]  ── UNTRUSTED
```

| # | Threat | Mitigation | Issue AC |
|---|---|---|---|
| T1 | Command writes outside workspace (`~/.ssh`, `/etc`) | Kernel-enforced deny (SBPL/Landlock); userspace containment is defense-in-depth only | engine + policy issues |
| T2 | Command exfiltrates data over network during rehearsal | Default netns-loopback / seccomp / SBPL deny; `--allow-network` is explicit | network issues |
| T3 | Sandbox fails to init but command still runs | Fail-closed: `SANDBOX_INIT_FAILED`, never run unsandboxed | select/prepare issues |
| T4 | Symlink/`..`/TOCTOU escapes the writable root | Canonicalize + kernel enforcement at real path; overlay mounts at path; copy engines resolve within shadow; tests for symlink-out, `..`, race | path + engine tests |
| T5 | Resource exhaustion (fork bomb, disk fill, infinite loop, huge output) | pid namespace (overlay) / process reaping; disk+inode budget monitor aborts; wall-clock timeout; bounded output capture | monitor issue |
| T6 | Malicious diff content spoofs the UI (ANSI, huge lines, control chars) | Sanitize/escape control sequences in rendered paths+diffs; cap line length; never emit attacker-controlled raw ANSI | human renderer issue |
| T7 | Secret contents leak into the JSON report / logs | Redaction rules for known secret paths/patterns → `Redacted`; never log file bodies; report has no raw env dump | changes/classify issue |
| T8 | Approval confusion (user approves thinking it's still rehearsal) | Explicit gate copy; real run clearly announced; `--allow-network` echoed; drift check before real run | decision-gate issue |
| T9 | Command spawns background/daemon that survives rehearsal | pid-namespace reaping (overlay); process-group kill on all engines at teardown | execute/cleanup issues |
| T10 | Engine mis-selection silently weakens isolation | Selection is fail-closed; forced engine that is unusable errors; report always states engine + caveats | select issue |

### 10.3 Boundary-by-boundary expectations

Every externally reachable / parsing / persistence boundary has explicit rules:

- **CLI/arg boundary**: no shell injection by dryrun itself — argv form never passes
  through a shell; shell form uses `sh -c` with the user's exact string (the user's own
  content, not concatenated by us). Reject ambiguous flag/command splits.
- **Config file parser**: `deny_unknown_fields`; size-limited; never executes content;
  path values validated by the containment rules; a malformed config is a `CONFIG_ERROR`,
  never a panic.
- **Filesystem scan boundary**: bounded by `max_files`; follows no symlink out of the
  shadow; handles cycles; never reads a file body larger than the diff cap; treats special
  files (fifos, sockets, devices) as metadata-only.
- **Network boundary (rehearsal)**: external IP egress (`AF_INET`/`AF_INET6`/`AF_PACKET`)
  is default-denied in-kernel; local IPC (`AF_UNIX`/`AF_NETLINK`) and loopback (where the
  engine provides it) are allowed for tool compatibility (ADR-003). The portable engine's
  allowance of local IPC leaves a documented residual (a host-local daemon reachable over a
  unix socket); the overlay engine's netns removes it. `--allow-network` widens to full
  outbound and is recorded in the report.

- **Reporting-fidelity note**: kernel primitives (Landlock EACCES, netns unreachability,
  seccomp `RET_ERRNO`) *enforce* the deny but do not guarantee a complete per-attempt log.
  v1 reports blocked writes / network attempts **best-effort**; every report carries a
  caveat that the deny is kernel-enforced while enumeration may be incomplete. Do not claim
  completeness in any user-facing text.
- **Sandbox/kernel boundary**: all `unsafe`/FFI concentrated in `engine/macos/ffi.rs` and
  the linux syscall helpers, each with `// SAFETY:`; failures fail-closed.
- **Report/persistence boundary**: run directory created `0700` under a per-user base
  (`$XDG_STATE_HOME/dryrun` or `$TMPDIR`), never world-readable; cleaned on teardown;
  reports written with restrictive mode; no secrets in filenames.
- **Real-exec boundary**: runs with the user's normal privileges (no elevation), the same
  argv/shell string, after explicit approval + drift check.

### 10.4 Secret handling

- dryrun never creates, stores, or requires credentials. It has no network of its own
  (zero telemetry / no update check by policy).
- Redaction: file contents at known secret paths (`**/.ssh/*`, `**/.aws/credentials`,
  `**/.netrc`, `**/*.pem`, `**/id_*`, `.env*`) or matching secret regexes are recorded as
  `Redacted` in diffs; only metadata (path, size, mode-change) is shown. Redaction rules
  are configurable but on by default.
- Logs never contain file bodies or environment variable values.

### 10.5 Abuse cases (misuse of the tool)

- **A1 Using dryrun to *safely test* an exploit before firing it**: out of scope to
  prevent, but network default-deny means a rehearsal cannot complete a remote attack; the
  real run is the user's own responsibility (same as running the command directly).
- **A2 `--writable /` / disabling protection**: dangerous-root deny list + `--writable-
  force` friction; documented warnings; report shows the widened scope prominently.
- **A3 Feeding dryrun a config from an untrusted repo**: config cannot execute code, is
  schema-validated, and cannot widen writable roots to dangerous locations without the
  force flag; `--no-config` and explicit `--config` exist. Document "review a repo's
  `.dryrun.toml` before trusting it," and do NOT auto-load project config that escalates
  privilege silently (project config may only *narrow*, never *widen*, dangerous scope —
  see config issue).

### 10.6 Dependency & supply-chain risk

- `cargo-deny` in CI: license allowlist, RustSec advisory gate, ban unmaintained/duplicate.
- `Cargo.lock` committed; CI + release build `--locked`.
- Minimal dependency set (research/02); `unsafe` only in named modules.
- Release: build in CI from a tagged commit; publish checksums (SHA-256) for each artifact;
  provide SLSA-style provenance / build attestation where GitHub Actions supports it;
  document verification in the release notes. (Signing keys are generated by the user, not
  the agent — see release issue handoff.)
- `SECURITY.md` with a private disclosure channel and response expectations.

### 10.7 Secure defaults recap

Network denied · writes workspace-only · fail-closed sandbox · no telemetry · run dir
`0700` · redaction on · dangerous `--writable` roots refused · non-TTY never auto-runs the
real command without `--yes`.

---

## 11. Failure modes & handling

| Condition | Detection | Behavior | Exit |
|---|---|---|---|
| Forced engine unusable | probe | error with reason | 40 |
| No usable engine | probe all | error + remediation (doctor hint) | 41 |
| Seatbelt/namespace/landlock init fails | prepare/execute | fail-closed, cleanup | 42 |
| Overlay mount EPERM (userns blocked) | prepare | auto-fallback to portable (or error if forced) | (fallback) |
| Timeout | monitor | kill process group, mark `limit_hit=timeout` | 43 |
| Disk/inode budget exceeded | monitor | kill, mark limit, partial changeset flagged | 43 |
| Output cap exceeded | tee | stop capturing, mark truncated, keep running to limit | (run continues) |
| Rehearsed command non-zero | wait | still render changeset (side effects up to failure) | 10 |
| Blocked write/network + violation_mode=Fail | scan | render + fail | 30 |
| Drift detected pre-real-run | drift check | warn; TTY re-confirm; `--yes` proceeds with note | (per choice) |
| SIGINT during rehearsal | signal handler | kill child group, cleanup, exit | 130-style→map |
| Panic | Drop guard | cleanup runs, `INTERNAL_ERROR` | 70 |

---

## 12. Configuration file

- Discovery order (unless `--config`/`--no-config`): `$DRYRUN_CONFIG`, then
  `./.dryrun.toml` (project, may only narrow), then `$XDG_CONFIG_HOME/dryrun/config.toml`
  (user). Merge: user config sets defaults; project config may tighten (reduce writable
  scope, force network deny) but not loosen dangerous scope; flags override both.
- Format TOML, `deny_unknown_fields`. Example:

```toml
network = "deny"                 # deny | allow
timeout = "120s"
max_output = "10MiB"
max_disk = "1GiB"
max_files = 100000
writable = ["./"]                # relative to config location; validated
redact = true                    # secret redaction on
pager = true
```

- Any parse/validation error → `CONFIG_ERROR` with line/context, never a panic.

---

## 13. Storage / run-directory layout

Per rehearsal, a run dir under `${XDG_STATE_HOME:-$HOME/.local/state}/dryrun/runs/<id>` on
Linux and `${TMPDIR}/dryrun/runs/<id>` on macOS, mode `0700`:

```
<run>/
  overlay/            # linux-overlay: {upper,work,merged} mount points
  shadow/             # macos/portable: cloned/copied workspace
  tmp/                # private tmp mapped into the sandbox
  manifest.json       # pre-run snapshot (path→Meta) for copy engines
  report.json         # last report (if persisted)
  stdout.cap, stderr.cap  # bounded captures
```

Cleaned up on teardown unless `--keep-run-dir` (debug flag, documented, off by default).
Never placed in the workspace (so the tool never pollutes what it measures).

---

## 14. Observability

- Logging via a light `tracing`-free logger (or `log`+`env_logger` — pick minimal in
  scaffold) to **stderr**; `-v/-vv` raise level; `--json` keeps stdout pure.
- No metrics/telemetry leave the machine (policy).
- `doctor` prints: OS, kernel/macOS version, engine probes (usable/degraded/reasons),
  APFS/reflink status, userns/AppArmor status, Landlock ABI, remediation steps.

---

## 15. Testing strategy (summary; full matrix in ISSUE_PLAN §validation)

- Unit: policy/path containment, config precedence, changeset classification, diff,
  report serialization (snapshot), exit-code mapping.
- Engine integration (os-gated): create/modify/delete/rename/mode/symlink capture;
  out-of-workspace write denied+reported; network denied (and allowed with flag);
  resource limits enforced; fail-closed on forced-unusable engine.
- Security regression tests: symlink-escape, `..`-escape, fork bomb reaped, disk budget,
  ANSI-injection in diff sanitized, secret redaction, dangerous `--writable` refused.
- CI matrix: macOS (APFS) + Linux (a userns-enabled kernel *and* a userns-restricted
  configuration to exercise the portable fallback).
- Golden files: `--help`, man page, JSON schema, SBPL profile, sample reports.

---

## 16. Glossary

- **Workspace**: the writable, captured root (default cwd).
- **Shadow**: a throwaway copy/clone of the workspace the rehearsal writes into (copy
  engines).
- **Overlay upperdir**: the overlayfs layer holding rehearsal writes (overlay engine).
- **ChangeSet**: the structured diff of what the rehearsal would do.
- **Engine**: a concrete sandbox implementation behind `SandboxEngine`.
- **Rehearsal**: the sandboxed run. **Real run**: the post-approval unsandboxed run.
