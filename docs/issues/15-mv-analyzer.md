# Title

mv analyzer

## Summary

Implement the mv-family analyzer: into-dir vs pairwise destination resolution, `Move` entries for sources, `Overwrite` entries for clobbered targets, `-n`/`-i` semantics.

## Context

DESIGN §6.6. mv is the only analyzer producing `Move` dispositions (restored by rename-back, §9.3) and the pairing of Move+Overwrite for clobbering moves.

## Scope

- `src/analyze/cmd/mv.rs` using the shared trait/harness from 14.
- Flags: BSD `mv` booleans `-f -i -n -v -h`; `gmv` additionally long forms and valued `-t DIR` / `--target-directory=DIR` (sets DEST), `-S SUFFIX` / `--suffix=` (consume value), `--backup[=…]`, `-b`, `--no-clobber`, `--interactive`, `--force`, `--update[=…]`.
- Destination resolution per §6.6: DEST = last operand (or `-t` value); `stat` (follow symlink) → existing dir ⇒ into-dir mode (`TARGET = DEST/basename(SRC)` per SRC); else pairwise (single SRC only; incoherent forms → plan only coherent pairs, note verbose).
- Entries per §6.6:
  - existing SRC → `Move { dest: TARGET }`, payload MetadataOnly, kind from lstat;
  - existing TARGET (lstat) && !`-n`/`--no-clobber` → `Overwrite`, payload Content.
- Dedup with strength rule across the command.

## Detailed Requirements

1. Table tests ≥ 25 cases: `mv a b` (b absent → Move only); `mv a b` (b exists → Move + Overwrite pair); `mv -n a b` (b exists → Move only); `mv a b c dir/`; `mv a dir` (dir symlinked-to-dir → into-dir via stat-follow); `mv *.txt dir/`; `gmv -t dir a b`; `mv a` (usage-incoherent → empty); `mv $X b` (partial); `mv 'sp ace' b`; trailing-slash DEST absent (pairwise, target parent may not exist — Move planned anyway; rename may fail, harmless F7); SRC == TARGET self-move (skip entry, verbose); case-only rename on case-insensitive APFS (single Move entry; F14 comment).
2. Cross-device note: plan shape identical (F17) — covered by a ramdisk test in 31, not here; add a code comment.
3. `Overwrite` target entry must record TARGET's lstat kind (file vs dir — moving onto a dir errors in mv itself; still plan Overwrite only for non-dir targets; dir targets in pairwise mode → into-dir already handled).

## Acceptance Criteria

- [ ] All table cases produce exact expected entry sets (path, disposition, move_dest, kind).
- [ ] Move+Overwrite pairing verified: `mv a b` with existing b yields exactly 2 entries with the strength/dedup invariant intact.

## Validation

`cargo test analyze::cmd::mv` green.

## Dependencies

13, 14 (trait/harness).

## Non-goals

Actual rename-back logic (24/25); cp (16).

## Design References

DESIGN.md §6.6, §9.3; §13 F14/F17.
