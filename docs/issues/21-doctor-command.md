# Title

`undo doctor`

## Summary

Implement the environment/health check command with the DESIGN §3.6 check table, human table output and `--json`.

## Context

Doctor is the user's first debugging surface and the install-verification step in README and acceptance tests. Checks reuse primitives from store (05), config (04), and the clone-capability probe (07).

## Scope

- `src/doctor.rs` implementing every §3.6 row:
  1. binary on PATH & shadowing (`which -a`-equivalent via PATH scan; compare to `current_exe`);
  2. hook installed (search `~/.zshrc` for `undo init zsh`; also `$ZDOTDIR/.zshrc` when set);
  3. hook active (`UNDO_SESSION` present in env);
  4. zsh version ≥ 5.8 (`zsh --version` parse; zsh missing → fail);
  5. store perms/owner (reuse 05's verifier in report-mode instead of refuse-mode);
  6. store volume `VOL_CAP_INT_CLONE` (07's probe) → warn-only when absent;
  7. TM exclusion xattr present → warn + `re-run: undo doctor --fix-tm` (implement `--fix-tm` re-applying the xattr; the ONLY mutating doctor flag);
  8. free space > 1 GiB on store volume (statfs);
  9. config parses (surface first error);
  10. FDA hint: attempt `readdir ~/Desktop` → EACCES/EPERM ⇒ info-level hint text about granting Full Disk Access to the terminal.
- Output: aligned table `CHECK | STATUS(ok/warn/fail/info) | DETAIL`; summary line; exit 0 when no `fail`, exit 1 otherwise.
- `--json`: `{"format":1,"checks":[{"id","status","detail"}],"ok":bool}` stable ids (snake_case fixed strings listed in code as constants).

## Detailed Requirements

1. Every check must be individually testable: `Check` trait objects with injected environment (fake PATH, fake HOME via env override in tests).
2. Doctor never mutates anything except under explicit `--fix-tm`.
3. Missing store = not an error (fresh install): store checks report `info: store not yet created (created on first snapshot)`.
4. Check ids (fixed): `binary_path`, `hook_installed`, `hook_active`, `zsh_version`, `store_perms`, `volume_clone`, `tm_exclusion`, `disk_free`, `config_parse`, `fda_hint`.

## Acceptance Criteria

- [ ] On a healthy dev machine setup: exit 0, all ok/info.
- [ ] Each failure path unit-tested with injected env (10 tests minimum, one per check).
- [ ] `--json` golden test; ids stable.
- [ ] `--fix-tm` applies the xattr and turns the check green.

## Validation

`cargo test doctor` green; manual run screenshot/output in PR.

## Dependencies

04, 05, 07 (probe), 20 (hook artifacts to detect).

## Non-goals

Auto-fixing perms (ADR-006 forbids), network checks (none exist), macOS version gating.

## Design References

DESIGN.md §3.6, §12.6; ADR-006; research/macos-platform-notes.md §6.
