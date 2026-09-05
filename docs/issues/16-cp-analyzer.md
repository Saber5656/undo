# Title

cp analyzer

## Summary

Implement the cp-family analyzer: overwrite-only planning for single-file and `-R` into-dir forms, with bounded source walks to discover clobbered destination paths.

## Context

DESIGN §6.6. cp never deletes; it only overwrites — the plan captures pre-images of destination files that will be clobbered. The `-R` case requires walking the *source* to predict destination collisions, bounded by §7.5 caps.

## Scope

- `src/analyze/cmd/cp.rs` (shared trait/harness).
- Flags: BSD booleans `-a -f -i -n -p -v -X -c -L -P -H -R -r`; `gcp` long/valued forms: `-t DIR`, `--target-directory=`, `-S/--suffix=`, `--backup[=…]`, `--no-clobber`, `-u/--update`.
- Destination resolution identical to mv (§6.6, reuse the helper extracted from 15).
- Entries:
  - single-file mode: TARGET exists (lstat) && !`-n` → `Overwrite/Content`;
  - into-dir non-recursive: per SRC, `DIR/basename(SRC)` exists → `Overwrite/Content`;
  - `-R`/`-r` into-dir: walk each SRC dir (bounded by `max_entries_per_op` share + deadline; reuse `snapshot::Budget` type): for each source *file* node, if corresponding `DIR/rel/path` exists (lstat) → `Overwrite/Content` entry. Cap exhaustion → `partial: true` + `too large` notice.
- `-n` → no entries at all.

## Detailed Requirements

1. Table tests ≥ 20 cases: `cp a b` (b absent → empty plan, no op); `cp a b` (b exists); `cp -n a b` (empty); `cp a b dir/`; `cp -R src dst` (dst absent → empty — cp creates fresh tree); `cp -R src existing-dst` with partial collisions (only colliding files planned); `cp -R src dst` where dst/inner is a symlink to elsewhere (collision detection must lstat, not follow — entry records the symlink itself); budget-exhaustion case (tiny cap); `gcp -t dir a`; `cp $X y` partial; `cp a a` self (skip, verbose).
2. The source walk must not follow symlinks and must never write (grep gate reused).
3. cp's own symlink flags (`-L -P -H`) do NOT change our planning (we snapshot destination pre-images regardless) — comment + one test documenting this conservative equivalence.

## Acceptance Criteria

- [ ] All table cases pass with exact entry sets.
- [ ] Collision walk on a 1,000-file fixture with cap 50 → ≤ 50 planned entries + partial notice.

## Validation

`cargo test analyze::cmd::cp` green.

## Dependencies

13, 14 (harness), 15 (dest-resolution helper).

## Non-goals

Restore semantics; mv (15); preserving cp's own attribute behavior.

## Design References

DESIGN.md §6.6, §7.5.
