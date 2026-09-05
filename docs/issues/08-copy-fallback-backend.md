# Title

Bounded copy fallback backend

## Summary

Implement the cross-volume/non-cloneable fallback: streamed byte copy with a per-operation byte budget, xattr preservation, mode/mtime preservation, and clean partial-failure semantics.

## Context

Used when `clonefile` returns EXDEV/ENOTSUP (external volumes, non-APFS). DESIGN §7.4 defines the budget behavior: over-budget entries become `not_protected_reason: "over_size_cap"` and the op continues.

## Scope

- `src/snapshot/copy.rs`: `fn copy_file(src, dst, budget: &mut ByteBudget) -> Result<CopyOutcome>`:
  - open src with `O_NOFOLLOW`; stream in 1 MiB chunks; check budget per chunk; over budget → delete partial dst, return `CopyOutcome::OverBudget`.
  - preserve mode (`fchmod`), mtime (`futimens`), xattrs (via `copyfile(3)` with `COPYFILE_XATTR` after content copy, or manual listxattr/getxattr/setxattr loop — choose one, document; ACLs explicitly not preserved (F16)).
- `ByteBudget` from `limits.max_copy_bytes` (config), shared across the whole operation.
- Integration hook for the walk (07): fallback closure signature agreed with issue 10 (`Fn(src, dst, &mut ByteBudget) -> …`).

## Detailed Requirements

1. Budget accounting counts bytes actually written (not st_size), so sparse tails don't over-charge.
2. ENOSPC mid-copy → delete partial dst, return abort-class error (op abort per F1, handled in 10).
3. Symlinks are never followed at either end (`O_NOFOLLOW`; dst is always a fresh name inside staging).
4. Copy of a 0-byte file, a file with 2 xattrs, and a >budget file each covered by tests.
5. No fsync per payload file (DESIGN §7.6 rationale: fail-open economics).

## Acceptance Criteria

- [ ] RAM-disk (non-clone volume w.r.t. store) file copies succeed with mode+mtime+xattr preserved (mtime tolerance 1 s).
- [ ] Budget 1 MiB against a 2 MiB file → `OverBudget`, no dst file remains, budget not corrupted for subsequent small file.
- [ ] ENOSPC simulation (tiny RAM disk as *destination* store) → abort-class error, staging clean.

## Validation

`cargo test snapshot::copy` on CI with `scripts/mk-ramdisk.sh` fixtures.

## Dependencies

04, 05 (07's ramdisk helper shared).

## Non-goals

Clone path (07), selection logic (10).

## Design References

DESIGN.md §7.2, §7.4, §13 (F1, F16); ADR-002.
