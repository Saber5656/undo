# Title

Restore executor with inverse operations

## Summary

Execute restore plans: inverse-op snapshot first, then ordered application (rename-backs, payload materialization with symlink-safe writes, metadata application), per-entry result collection, and the restore-op manifest record.

## Context

DESIGN §9.1 state machine (SNAPSHOT-INVERSE → EXECUTE → RECORD → REPORT); §12.4 write-safety rules are hard security requirements; §8.4: executor holds a shared lock for the whole run so GC cannot evict the op mid-restore.

## Scope

- `src/restore/exec.rs`: `fn execute(store, manifest, plan) -> RestoreReport`.
- Take SH lock (05) for the duration; verify op dir still exists after acquiring (GC race, F9).
- Inverse op: build plan from 24's `inverse_targets`, run through the snapshot orchestrator (10) with `OpContext { kind: Restore, restores: op_id, command_text: "undo <op-id>" }`; inverse snapshot failure (ENOSPC etc.) → **abort the whole restore** before any write (safety: never restore without the escape hatch), exit-1-class error. Empty inverse plan is fine (nothing will be replaced).
- Apply actions in plan order:
  - move-back: `rename(move_dest, path)`; EXDEV → copy+delete fallback via payload-less copy of the moved file (use 08's copy with a fresh budget) — record outcome;
  - payload materialization (overwrite/delete restores): clone-first (07) from payload into a temp name inside the **verified parent**, then `rename` into place; parent verification per §12.4: walk components from the nearest recorded ancestor with `O_NOFOLLOW|O_DIRECTORY` `openat`; mismatch → entry failure `parent path changed (symlink?)`;
  - directory payloads: materialize tree (dirs with recorded modes, files cloned, symlinks recreated);
  - metadata: `chmod`/`chown`(uid,gid) to recorded values via `fchownat(AT_SYMLINK_NOFOLLOW)`; EPERM on chown → entry failure with `requires ownership privileges; not attempted with sudo`;
  - `--into` materialization: same write-safety inside DIR.
- Per-entry results: `Restored | Failed(reason) | Skipped(reason)`; report aggregates → exit 0 all-restored, exit 4 any failure (§3.3), summary line counts.
- RECORD: inverse op was already committed pre-execution (it IS the record); append restore outcome into the inverse op manifest? NO — manifests are immutable post-commit (ADR-004); the `RestoreReport` is CLI-output only. Add a code comment.

## Detailed Requirements

1. Every write path goes through the verified-parent helper; add an adversarial test: symlink planted at the recorded parent after snapshot → entry fails, nothing written outside (assert target volume untouched via before/after tree hash of the attack destination).
2. Restore of an op whose payload dir was tampered to include `payload/../x` refs is impossible by 06 validation — assert load-level rejection test here as a belt-and-suspenders integration case.
3. Round-trip integration tests (fixtures via orchestrator, then real `rm`/`mv`/`cp`/`sed -i`/`chmod` shell-outs, then execute restore): content byte-equality (hash), mode/uid/gid equality, symlinks preserved, xattr survival (clone path); each analyzer family gets one round-trip here (full matrix lives in 31).
4. Restore-of-restore: run `undo` twice (restore then restore the inverse) → filesystem returns to post-command state (test).
5. Partial failure: one entry sabotaged (parent replaced) among three → exit 4, other two restored, report lists all three with statuses.

## Acceptance Criteria

- [ ] Round-trips green for all five families; restore-of-restore green.
- [ ] Symlink-parent attack test: no write escapes; entry failed; exit 4.
- [ ] Inverse-op-first invariant: sabotage inverse snapshot (read-only store mid-test) → no user file modified.
- [ ] SH-lock held: concurrent `gc` (spawned process) cannot evict the restoring op (test with aggressive retention).

## Validation

`cargo test restore::exec` green on CI.

## Dependencies

07, 08, 10, 24.

## Non-goals

Prompting/TTY UX (26), GC internals (27).

## Design References

DESIGN.md §9.1, §9.3–§9.4, §8.4, §12.4, §13 F7/F9; ADR-002; ADR-004.
