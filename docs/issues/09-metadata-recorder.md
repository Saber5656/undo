# Title

Metadata recorder (lstat capture, recursive walk)

## Summary

Implement metadata-only capture for `chmod`/`chown` (and `mv` move-entries): single-path lstat records and the capped recursive variant for `-R` operations, producing manifest-ready attribute sets without payloads.

## Context

DESIGN §6.8 plans produce `Metadata`/`MetadataOnly` entries; §7.5 caps bound recursive walks (F5: `chown -R` on a million files must degrade gracefully).

## Scope

- `src/snapshot/meta.rs`: `fn record_meta(path) -> MetaRecord` (lstat: kind, mode, uid, gid, mtime, size for files) and `fn record_meta_recursive(root, budget) -> Vec<MetaRecord>` walking with the same `openat` safety rules as issue 07 (share the traversal helper — refactor 07's walker to accept a visitor if needed).
- Symlinks recorded as themselves (`lstat`), never followed (chmod `-h` semantics, §6.8).
- Budget: shares `max_entries_per_op` counter and soft deadline with the op-wide budget type (issue 07's `budget` — unify into `snapshot::Budget`).
- TCC/EACCES on a node → per-node `walk-error:<errno>` record (F20), walk continues.

## Detailed Requirements

1. `MetaRecord` fields align 1:1 with manifest entry attrs (§8.2) so issue 10 can map without conversion logic.
2. Recursive order: parent recorded before children (restore applies top-down; §9.3).
3. Budget exhaustion returns partial vec + `stopped_reason` (same shape as 07's `TreeReport` — shared type).
4. Unit tests: single file, dir tree with symlink + FIFO, permission-denied subdir (chmod 000 in test, restore after), cap exhaustion on a 1,000-node tree with budget 100.

## Acceptance Criteria

- [ ] Records match `stat -f` ground truth for mode/uid/gid on fixtures.
- [ ] Permission-denied subtree yields walk-error records and does not abort the walk.
- [ ] Budget-100 walk returns exactly ≤ 100 records + stopped_reason.

## Validation

`cargo test snapshot::meta` green on CI.

## Dependencies

04 (budget values), 07 (shared traversal/budget types — coordinate; if 07 not merged, land the shared type here and 07 rebases).

## Non-goals

Restoring metadata (25); analyzer flag parsing (18).

## Design References

DESIGN.md §6.8, §7.5, §8.2, §12.4, §13 (F5, F20).
