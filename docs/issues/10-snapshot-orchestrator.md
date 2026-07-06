# Title

Snapshot orchestrator: plan → committed operation

## Summary

Implement the engine that takes a `SnapshotPlan` and produces a committed operation: backend selection per entry (clone vs copy vs metadata-only), staging lifecycle, caps/deadline enforcement, warn-and-proceed error policy, and auto-GC trigger hook.

## Context

This is the integration point of issues 06–09 and the direct callee of `undo __hook` (19) and the restore engine's inverse-op capture (25). Behavior under failure is normative (DESIGN §7, §13 F1/F3/F4, ADR-005).

## Scope

- `src/snapshot/orchestrator.rs`: `fn execute(store, plan: SnapshotPlan, ctx: OpContext) -> SnapshotOutcome`.
- `SnapshotPlan`/`PlanEntry` types (define here; analyzers in wave 2 produce them): entries with `{path, disposition, payload_class}` per DESIGN §6; `OpContext {command_text, cwd, session, kind, restores}`.
- Flow per DESIGN §7.6: skip empty plans (no op); create staging via 06; per entry:
  - `MetadataOnly` → 09 recorder (recursive iff dir + metadata disposition);
  - content file/symlink/dir → 07 walker/clone with 08 fallback closure on EXDEV/ENOTSUP;
  - store-path guard: entries under the store root are dropped with a verbose notice (§12.4 self-reference guard — second line of defense after 13).
- Error policy mapping (ADR-005): ENOSPC/abort-class → abort op (delete staging best-effort), warn `store unavailable`/`too large`, return `SnapshotOutcome::Aborted`; per-entry walk-errors → entry recorded with `not_protected_reason`, op `partial: true`; deadline/cap → stop, keep captured entries, `partial` or `aborted` per whether anything useful was captured.
- Compute `logical_bytes`, populate manifest (06), commit, append index, then call `lifecycle::auto_gc_check(store)` (stub until 27; define the fn signature now, no-op body with TODO(issue-27)).

## Detailed Requirements

1. Dedup + disposition-strength rule applied here as a final invariant check (analyzers should already dedup; orchestrator asserts): `Delete > Overwrite > Move > Metadata` (§6.6).
2. Backend selection: `entry.st_dev == store.st_dev` → clone-first; else copy; clone ENOTSUP/EXDEV at file level falls to copy transparently (budget applies only to copied bytes).
3. Manifest `aborted`/`partial`/`notices` fields must reflect exactly what happened (E2E greps these).
4. Never touches any user file for writing — capture-only (assert in tests via mtime comparison of sources).
5. Integration tests build plans directly (no parser needed): delete-file, delete-tree, overwrite-file, move+overwrite pair, metadata-recursive, mixed-volume plan (ramdisk), ENOSPC abort, cap abort, deadline abort (deadline 100 ms + slow tree).

## Acceptance Criteria

- [ ] Each integration scenario yields the exact manifest shape defined in DESIGN §8.2 (golden-file compare with normalized ids/timestamps).
- [ ] ENOSPC leaves no committed op and no staging residue (best-effort verified).
- [ ] Mixed-volume plan: same-volume entry cloned (wall-time bound), ramdisk entry copied (budget charged).
- [ ] Source files' content and mtimes untouched by snapshotting.

## Validation

`cargo test snapshot::orchestrator` green on CI including ramdisk scenarios.

## Dependencies

06, 07, 08, 09.

## Non-goals

Parsing (wave 2), notices wording for parser-level skips (12/13), GC internals (27).

## Design References

DESIGN.md §6 (plan shape), §7, §8.2, §13 (F1, F3, F4); ADR-002; ADR-005.
