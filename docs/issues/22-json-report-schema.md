# 22 — JSON report emitter and committed schema

## Title
JSON report emitter and committed schema

## Summary
Implement the `report` module that serializes a run into the versioned JSON object in
DESIGN §7, commit a `docs/report.schema.json`, and snapshot-test both so the machine
contract (used by agents) cannot drift silently.

## Context
The JSON report is the stable contract for automation (DESIGN §1.3, §7). `schema_version`
governs compatibility; consumers ignore unknown fields.

## Scope
- In: `src/report/mod.rs` (`Report` struct + assembly), `src/report/json.rs` (emit),
  `docs/report.schema.json` (committed JSON Schema), snapshot tests.
- Out: human rendering (23); decision gate (24).

## Detailed Requirements
1. `Report` struct mirroring DESIGN §7 exactly: `schema_version` (`"1.0"`), `meta`
   (tool_version, os, engine, engine_selection, started_at RFC3339, duration_ms, command,
   policy), `exit` (phase/code/name/child_exit/signal/limit_hit), `changes` (from the
   `ChangeSet`), `summary`.
2. `assemble(policy, selection, outcome, changeset, timing) -> Report` builds it from the
   pipeline outputs. Timestamps live only in `meta` (keeps the rest deterministic, DESIGN
   §6.2).
3. `emit(report, writer)`: pretty or compact JSON; under `--json` write ONLY this object to
   stdout (logs to stderr); under `--output FILE` also/instead write to the file (mode
   0600).
4. Commit `docs/report.schema.json` (JSON Schema draft 2020-12) describing the object;
   fields required/optional per DESIGN §7; `additionalProperties` permitted (forward-compat)
   but documented. A test validates a produced report against the schema (use a schema
   validator crate as a dev-dependency, or a hand-rolled check if none is acceptable per
   research/02).
5. Snapshot test (`insta`) of a report built from a fixture `ChangeSet` (NoopEngine or a
   canned changeset), with the timestamp/duration fields normalized.
6. Versioning rules documented in-module: additive → minor bump; breaking → major (avoided
   in v1); consumers must ignore unknown fields.

## Acceptance Criteria
- A produced report validates against `docs/report.schema.json`.
- The snapshot (normalized timestamps) is stable across runs of the same fixture.
- `--json` output is exactly one JSON object on stdout with nothing else.
- All DESIGN §7 fields are present with the documented names/types.

## Validation
- `cargo test report::` (snapshot + schema-validation test).
- Manual: `dryrun -c 'true' --json | python3 -c 'import json,sys; json.load(sys.stdin)'`.

## Dependencies
02, 05, 06, 07, 08. (`assemble` consumes the exit mapping/`ReportExit` from 02, the policy
from 05, `ExecOutcome` from 06, the `SelectionReport` from 07, and the `ChangeSet` from 08.)

## Non-goals
No human/color output (23); no interactive gate (24).

## Design References
DESIGN §7 (schema), §6.2 (determinism), §1.3 (automation contract).
