# Review resolution addendum

- Repository: `Saber5656/dryrun`
- Pull request: #1
- Original PR head before this resolution addendum: `a0035133e7d43b0f2f2f03f4e5739d297b0479d3`
- Scope: the review findings listed below are converted into normative design contracts and focused verification gates.
- The immutable current PR head is supplied by the parent task's fresh GitHub read immediately before review/reply/resolve; any later head change invalidates this review evidence and requires a fresh review.
- This addendum records design-level handling only; it does not claim implementation, test, build, CI, or security validation is complete.
- Per task instruction, the PR review bot is not re-triggered after these responses/resolutions.

## 1. Thread `PRRT_kwDOTNkAbc6QDKDs` — Preserve rehearsal failure exit in preview mode

**Normative resolution**: Approval/no-execute policy is independent from child status: any non-zero rehearsal exit maps to `REHEARSAL_COMMAND_FAILED` (10) after rendering, even when execution is skipped or no approval channel exists.

**Focused verification gate**: Table-test zero/non-zero rehearsal exits with approval, `--no-execute`, and no approval channel; assert the report is rendered but the non-zero case always returns 10.

**Completion boundary**: this section is a contract for the later implementation/full-validation gate, not evidence that that gate has already passed.

## 2. Thread `PRRT_kwDOTNkAbc6QDKDt` — Specify safe capture for extra writable roots

**Normative resolution**: For v1, `--writable` roots outside the workspace are rejected by the `linux-overlay` engine unless each root has its own captured shadow/overlay and scan identity. No path is added to Landlock while writes would still reach the real outside directory; forced unsupported use fails closed.

**Focused verification gate**: Test repeated in-workspace and outside roots, engine selection, forced-engine behavior, and an attempted write; assert unsupported outside roots are rejected before execution and supported roots have isolated captures.

**Completion boundary**: this section is a contract for the later implementation/full-validation gate, not evidence that that gate has already passed.

## 3. Thread `PRRT_kwDOTNkAbc6QDKDu` — Represent changes outside the workspace root

**Normative resolution**: The v1 schema constrains writable roots to workspace descendants, so every `FileChange.path` remains relative to one root. Future multi-root support must add a stable `root_id` plus root-relative path and update report/rendering contracts before being enabled.

**Focused verification gate**: Pass an outside root and assert a clear validation error; for descendant roots assert paths are unambiguous, normalized, and cannot escape with `..`.

**Completion boundary**: this section is a contract for the later implementation/full-validation gate, not evidence that that gate has already passed.

## 4. Thread `PRRT_kwDOTNkAbc6QDKDv` — Keep the dryrun supervisor out of overlay namespaces

**Normative resolution**: Namespace setup and rehearsal execution run in a forked helper; the supervisor remains in the caller's namespace and owns the status pipe, drift check, and any later real-run decision.

**Focused verification gate**: Instrument parent and helper namespace identities before/after rehearsal and assert the parent is unchanged, helper teardown is observed, and post-rehearsal checks run in the original environment.

**Completion boundary**: this section is a contract for the later implementation/full-validation gate, not evidence that that gate has already passed.

## 5. Thread `PRRT_kwDOTNkAbc6QDKDw` — Gate loopback setup on a private net namespace

**Normative resolution**: `bring_up_loopback` is called only after a private network namespace was actually created. If no private netns exists, the engine must not touch host networking and must either leave networking unavailable or fail according to the documented policy.

**Focused verification gate**: Test `--allow-network` with and without a created netns, including privileged and unprivileged paths; assert no host interface mutation occurs.

**Completion boundary**: this section is a contract for the later implementation/full-validation gate, not evidence that that gate has already passed.

## 6. Thread `PRRT_kwDOTNkAbc6QDKDx` — Require Landlock ABI support for truncation denial

**Normative resolution**: A Linux engine claiming write-denial integrity requires Landlock ABI 3 or a separately verified fail-closed mechanism for truncation; ABI 1/2 is not silently downgraded. If the requirement is unavailable, select a genuinely safe alternative or reject the run.

**Focused verification gate**: Mock/probe ABI 1, 2, and 3 and exercise `O_TRUNC`/`truncate(2)` against an out-of-workspace file; assert unsupported ABIs cannot proceed under the integrity guarantee.

**Completion boundary**: this section is a contract for the later implementation/full-validation gate, not evidence that that gate has already passed.

## 7. Thread `PRRT_kwDOTNkAbc6QDKDz` — Keep `--json --yes` from corrupting stdout JSON

**Normative resolution**: The CLI rejects the incompatible `--json --yes` combination unless a separately specified output channel preserves stdout purity; v1 uses rejection, so real command output can never be appended to machine-readable stdout.

**Focused verification gate**: Invoke the combination and assert a deterministic argument error with no non-JSON output; test `--json` preview and interactive/normal execution separately.

**Completion boundary**: this section is a contract for the later implementation/full-validation gate, not evidence that that gate has already passed.

## 8. Thread `PRRT_kwDOTNkAbc6QDKD0` — Treat newly created paths as drift candidates

**Normative resolution**: The rehearsal manifest records expected absence for created and rename-destination paths. The pre-execution drift check compares both existence and metadata, and flags an unexpected appearance before approval.

**Focused verification gate**: Create, delete, rename, and externally introduce paths between rehearsal and approval; assert created-path appearance and metadata drift are detected.

**Completion boundary**: this section is a contract for the later implementation/full-validation gate, not evidence that that gate has already passed.

## 9. Thread `PRRT_kwDOTNkAbc6QDKD1` — Align `--` synopsis with argv parsing

**Normative resolution**: `--` terminates dryrun options and preserves argv boundaries without shell parsing. Shell execution is available only through the explicitly documented `-c/--command` form.

**Focused verification gate**: Pass arguments containing spaces, shell metacharacters, and a literal program name after `--`; assert execve-style argv semantics and no implicit shell.

**Completion boundary**: this section is a contract for the later implementation/full-validation gate, not evidence that that gate has already passed.

## 10. Thread `PRRT_kwDOTNkAbc6QDKD3` — Prevent project configs from disabling redaction

**Normative resolution**: Untrusted project `.dryrun.toml` cannot set `redact=false` or otherwise weaken secret redaction. Only trusted user configuration, environment, or explicit CLI policy can weaken it, and the precedence is documented.

**Focused verification gate**: Run with project configs attempting to disable/narrow redaction and with trusted overrides; assert project input is ignored/rejected and `.env`/credential material remains masked.

**Completion boundary**: this section is a contract for the later implementation/full-validation gate, not evidence that that gate has already passed.

## 11. Thread `PRRT_kwDOTNkAbc6QDKD6` — Keep run dirs outside workspace even with env overrides

**Normative resolution**: After canonical resolution, the run directory must be outside every captured root. `XDG_STATE_HOME`/`TMPDIR` values that overlap a root are rejected or replaced with a safe fallback before shadow creation.

**Focused verification gate**: Set both variables to workspace descendants, symlinked descendants, and an external directory; assert overlap is detected after realpath/canonicalization and no self-copy is possible.

**Completion boundary**: this section is a contract for the later implementation/full-validation gate, not evidence that that gate has already passed.

## 12. Thread `PRRT_kwDOTNkAbc6QDKD7` — Count changed files, not the whole shadow

**Normative resolution**: `max_files` counts the resulting changeset (new/modified/deleted entries), not the pre-existing copy/shadow inode population. A separate, explicitly named workspace-size guard may protect copy cost.

**Focused verification gate**: Use a large unchanged workspace and a small changed set, then a changeset over the limit; assert unchanged shadow files do not consume the changeset budget.

**Completion boundary**: this section is a contract for the later implementation/full-validation gate, not evidence that that gate has already passed.

## 13. Thread `PRRT_kwDOTNkAbc6QDKD8` — Detect xattr whiteouts in overlay upperdir

**Normative resolution**: Overlay scanning recognizes both character-device whiteouts and zero-length regular files carrying `user.overlay.whiteout` before classifying entries, so rootless/userxattr deletions become removals.

**Focused verification gate**: Construct both whiteout encodings and ordinary zero-length files; assert only the whiteouts produce deletions and ordinary files remain classified normally.

**Completion boundary**: this section is a contract for the later implementation/full-validation gate, not evidence that that gate has already passed.

## 14. Thread `PRRT_kwDOTNkAbc6QDKD-` — Reap daemonized children outside the process group

**Normative resolution**: The v1 guarantee is narrowed for engines without a PID namespace: T9 is claimed only when the platform reaper can account for descendants. Otherwise the command is rejected when strict cleanup is requested and documentation says process-group cleanup is best effort; no unconditional guarantee is made.

**Focused verification gate**: Run a child that calls `setsid()`/daemonizes on each engine, assert strict mode detects or rejects unsupported cleanup, and assert no success result claims a stronger guarantee than the engine provides.

**Completion boundary**: this section is a contract for the later implementation/full-validation gate, not evidence that that gate has already passed.

## 15. Thread `PRRT_kwDOTNkAbc6QDKEB` — Fall back when overlay prepare hits EPERM

**Normative resolution**: Overlay preparation returns a typed `EPERM`; if the engine was auto-selected, selection retries with the portable engine. A user-forced overlay engine fails with `SANDBOX_INIT_FAILED` and does not silently change engines.

**Focused verification gate**: Inject `EPERM` during the late mount/userns step and test auto-selected versus forced engine paths; assert the fallback and exit codes match the policy.

**Completion boundary**: this section is a contract for the later implementation/full-validation gate, not evidence that that gate has already passed.

## 16. Thread `PRRT_kwDOTNkAbc6QDKEC` — Preserve hardlink topology in shadow copies

**Normative resolution**: The copy engine preserves hardlink inode relationships in the shadow/manifest. If it cannot reproduce them, it rejects or explicitly caveats the workspace before execution; it never produces a misleading single-path changeset.

**Focused verification gate**: Create two hardlinked paths, mutate one in rehearsal, and assert both paths are represented; test an unsupported filesystem and assert fail-closed behavior.

**Completion boundary**: this section is a contract for the later implementation/full-validation gate, not evidence that that gate has already passed.

## 17. Thread `PRRT_kwDOTNkAbc6QDKEF` — Reject home as the workspace root

**Normative resolution**: The dangerous-root rule applies to the workspace root as well as `--writable`: `$HOME` and equivalent canonical roots are rejected by default, with any explicit force option separately documented and surfaced as a high-risk mode.

**Focused verification gate**: Run from/pass `$HOME`, symlinked home, `/`, and a normal project root; assert default rejection and explicit-force audit behavior.

**Completion boundary**: this section is a contract for the later implementation/full-validation gate, not evidence that that gate has already passed.

## 18. Thread `PRRT_kwDOTNkAbc6QDKEH` — Keep private `/tmp` under the monitored budget

**Normative resolution**: Private `/tmp` is bind-mounted to the monitored `run_dir/tmp` (or mounted with explicit disk/inode limits and included as a monitored mount). Hard-coded `/tmp` writes therefore consume the same enforced budget.

**Focused verification gate**: Write via `/tmp` until disk/inode limits are approached and assert the monitor sees the usage and terminates according to the budget policy.

**Completion boundary**: this section is a contract for the later implementation/full-validation gate, not evidence that that gate has already passed.

## 19. Thread `PRRT_kwDOTNkAbc6QDKEJ` — Block metadata writes Landlock does not mediate

**Normative resolution**: Linux integrity mode adds an enforcement layer for metadata/xattr operations or refuses to advertise out-of-workspace integrity when that layer is unavailable. `chmod`, `chown`, timestamp, and xattr changes are included in the policy boundary.

**Focused verification gate**: Attempt each metadata operation on an out-of-workspace file under every Linux engine; assert denial or fail-closed engine selection, and verify the report does not claim unsupported protection.

**Completion boundary**: this section is a contract for the later implementation/full-validation gate, not evidence that that gate has already passed.