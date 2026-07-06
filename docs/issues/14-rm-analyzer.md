# Title

rm analyzer

## Summary

Implement the rm-family analyzer: BSD/GNU flag handling, `--` terminator, operand → `Delete` plan entries with content payloads.

## Context

DESIGN §6.5. First and simplest analyzer; establishes the shared analyzer trait + test harness pattern that 15–18 reuse.

## Scope

- `src/analyze/cmd/rm.rs` implementing a shared trait `Analyzer { fn analyze(&self, cmd: &SimpleCommand, r: &Resolver) -> AnalyzerResult }` (define the trait + `AnalyzerResult { entries: Vec<PlanEntry>, partial, notices }` in `analyze/cmd/mod.rs` here).
- Flag rules per §6.5: boolean-only set (`-d -f -i -I -P -R -r -v -W -x` + long GNU forms for `grm`: `--force --recursive --dir --verbose --one-file-system --preserve-root --no-preserve-root --interactive[=…]`); `--` ends flag parsing; unknown `-x…` before `--` treated as flags; everything after → operands.
- Operands run through resolution (13); each surviving path → `PlanEntry { path, disposition: Delete, payload: Content }` with kind from lstat (file/dir/symlink/special).
- Snapshot regardless of `-r` presence for directories and regardless of `-i/-I` (§6.5).
- Dedup within the command (same path once).

## Detailed Requirements

1. Table tests ≥ 25 cases: `rm x`; `rm -rf dir`; `rm -- -weirdfile`; `rm -f *.log`; `rm nonexistent`; `rm symlink` (entry kind symlink, target untouched); `rm 'a b'`; `rm ~/x`; `rm $F y` (partial, y planned); `rm -i x`; `grm --force x`; `rm dir` without -r (still planned); `rm /abs/path`; mixed flags+`--`+dash-file; empty operand list (no entries, no op).
2. Fixture-based: each case sets up a temp tree, runs analyze, asserts exact entry set `{path, disposition, kind}`.
3. Analyzer must not stat targets more than once per operand (pass through resolution results; performance note §14).

## Acceptance Criteria

- [ ] All table cases green, including symlink non-following and `--` handling.
- [ ] Trait + harness reusable: harness helper `analyze_str("rm -rf x", fixture)` exists for 15–18.

## Validation

`cargo test analyze::cmd::rm` green.

## Dependencies

13.

## Non-goals

Snapshot execution (10), other commands (15–18).

## Design References

DESIGN.md §6.5, §6.2 table; §13 F18/F19.
