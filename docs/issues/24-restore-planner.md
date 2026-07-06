# Title

Restore planner and conflict detection

## Summary

Implement the pure-logic restore planner: given a manifest and current filesystem state, produce an ordered restore plan with per-entry actions and the DESIGN §9.3 conflict taxonomy — no writes.

## Context

Separating planning (this issue) from execution (25) makes conflict logic table-testable against fixtures. The taxonomy table in §9.3 is normative; `--force`/`--into` variants change actions, not detection.

## Scope

- `src/restore/plan.rs`: `fn plan(manifest, mode: RestoreMode) -> RestorePlan`; `RestoreMode = Normal | Force | Into(PathBuf) | DryRun` (DryRun plans like Normal).
- `RestorePlan { actions: Vec<Action>, conflicts: Vec<Conflict>, skips: Vec<Skip> }` with `Action` ordered per §9.3: move-backs, overwrite-restores, delete-restores, metadata-restores; entry-index order within class.
- Current-state checks (all `lstat`):
  - move: `move_dest` exists? `path` occupied? → conflicts per table;
  - overwrite: never a conflict (inverse-op covers it); missing current file → still restorable (action notes `recreates`);
  - delete: `path` exists → conflict `AlreadyExists`;
  - metadata: `path` missing → `Skip::MissingPath` (per-entry, not a conflict).
- Force mode converts blocking conflicts into actions flagged `replaces_current: true` (executor inverse-snapshots them first); `move_dest missing` stays a per-entry failure marker (`Action::Failed` planned).
- Into mode: every content-payload entry → `Action::MaterializeInto { dst: DIR/<index>-<basename> }`; metadata entries skipped; no conflicts by construction.
- Inverse-plan derivation: `fn inverse_targets(plan) -> Vec<PlanEntry>` — the exact set of current paths EXECUTE will create/replace/re-attribute, shaped as a snapshot plan for 25 (delete-restore target that exists → Overwrite-style capture; move-back target path → its current occupant if any; metadata targets → Metadata entries).
- Corrupt/aborted-empty manifests refused upstream (06/26); planner asserts validated input.

## Detailed Requirements

1. Table tests ≥ 30 fixtures covering every §9.3 row × {Normal, Force}, plus Into on each disposition, plus mixed multi-entry op ordering (assert exact action order), plus F7 (post-failed-rm op: some paths still exist → delete-restore conflicts listed with count).
2. Planner is pure w.r.t. writes (lstat/readdir only — grep gate).
3. Conflict rendering strings (used by 26): each `Conflict` carries a one-line human description with escaped paths.

## Acceptance Criteria

- [ ] Every taxonomy cell has at least one passing test with exact expected plan.
- [ ] Ordering invariant verified on a mixed op (move+overwrite+delete+metadata).
- [ ] `inverse_targets` returns exactly the to-be-touched current paths for each mode (tested per mode).

## Validation

`cargo test restore::plan` green.

## Dependencies

06.

## Non-goals

Execution/writes (25), prompts (26), payload I/O.

## Design References

DESIGN.md §9.1–§9.4, §13 F7; ADR-004.
