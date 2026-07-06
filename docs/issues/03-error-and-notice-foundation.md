# Title

Error taxonomy, exit-code map, notice reporting

## Summary

Implement the shared `report` module: typed error enum mapped to the DESIGN exit codes, stderr notice emission with levels, control-character escaping for anything echoed to the terminal, and TTY/NO_COLOR-aware styling.

## Context

Every subsequent module returns these error types and emits notices through this module; the escaping rule is a security requirement (DESIGN §12.5: hostile filenames must not inject terminal escape sequences).

## Scope

- `src/report.rs` (+ `src/errors.rs` if cleaner): `UndoError` enum (thiserror) with variants at minimum: `Usage`, `Runtime(source)`, `ConflictsDetected`, `PartialRestore`, `StoreIntegrity(reason)`; `fn exit_code(&self) -> u8` implementing DESIGN §3.3 (2, 1, 3, 4, 5 respectively).
- `Notice` API: `notice_warn(reason: &str)`, `notice_verbose(...)`, gated by `ui.notices` level (`off|warn|verbose`) supplied by caller (config arrives in issue 04; take the level as a parameter now).
- All notices print to stderr as `undo: <text>`; not-protected notices as `undo: not protected: <reason>` (exact strings — E2E tests grep them).
- `fn escape_for_terminal(s: &str) -> String`: replaces C0 controls (except nothing — ALL C0 incl. `\n` when inline), C1, DEL with `\x{NN}` form; used by every code path printing paths or command text.
- Styling helper: bold/dim ANSI only when the target stream is a TTY and `NO_COLOR` unset and `ui.color != never` (level passed in). No color crates (ADR-003).
- Wire `main()`: top-level `Result<(), UndoError>` → process exit via the map; panics caught with a one-line `undo: internal error (bug): <msg>` to stderr, exit 1.

## Detailed Requirements

1. Exit-code mapping is exhaustive and unit-tested.
2. `escape_for_terminal` round-trip tested against: `"a\x1b]0;x\x07b"`, embedded newline, UTF-8 multibyte (must pass through unchanged), DEL.
3. Notice deduplication hook: a `NoticeSink` that can suppress repeats of the same reason within one process invocation (needed by ADR-005 rate-limiting; per-process is sufficient).
4. No `println!`/`eprintln!` outside this module by convention: add a clippy-visible comment and a grep check to the justfile (`just lint-report-usage`).

## Acceptance Criteria

- [ ] Unit tests cover every error variant → exit code, escaping cases above, notice level gating (off suppresses warn; verbose adds verbose-class).
- [ ] `undo list` stub (issue 01) now exits via the taxonomy (`Runtime` → 1) with `undo: `-prefixed stderr.
- [ ] A synthetic panic in a hidden test subcommand produces the internal-error line and exit 1 (test with `assert_cmd`).

## Validation

`cargo test report::` green; `just lint-report-usage` finds no stray eprintln outside `report.rs`.

## Dependencies

01.

## Non-goals

Config loading (04); localization; log files.

## Design References

DESIGN.md §3.3, §3.4, §6.4 (notice strings), §12.5; ADR-005.
