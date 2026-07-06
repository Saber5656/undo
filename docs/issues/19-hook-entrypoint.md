# Title

`undo __hook` entry point (analyze→snapshot pipeline)

## Summary

Implement the hidden `__hook` subcommand: read command text from stdin, run the full analysis pipeline (11→18), merge per-command plans into one operation (DESIGN §7.1), execute the snapshot (10), and honor the always-exit-0 fail-open contract.

## Context

This is the single integration point between the shell hook and the core (DESIGN §5.2). Its failure behavior is the product's most important reliability contract (ADR-005): nothing that happens here may disturb the user's command.

## Scope

- `src/cli.rs` + `src/hook_entry.rs`: `undo __hook --cwd <abs> --session <ulid> --hook-version <n>` reading stdin to EOF (cap 1 MiB + 64 KiB slack; beyond → treat as `Unsupported`, warn `could not parse command line`).
- Invalid UTF-8 stdin → lossy conversion (DESIGN §6.1 note).
- Pipeline: lex (11) → extract (12) → per matched command: resolve (13) + analyzer (14–18) → merge all `AnalyzerResult`s: concatenate entries, dedup by path with strength rule, `partial = any`, notices unioned (dedup by reason) → `SnapshotPlan` → orchestrator (10) with `OpContext { command_text, cwd, session, kind: Command, restores: None }`.
- Config gates: top-level `enabled = false` → immediate silent exit 0; `ui.notices` wiring to the sink (03).
- Hook-version mismatch (arg vs binary constant) → warn-level notice per affected invocation (stateless; the notice sink's per-process dedup applies), telling the user to re-`eval` the hook. Exact behavior per DESIGN §5.2.
- Fail-open enforcement: `__hook` wraps the entire pipeline in a catch-all: any `Err`/panic → optional warn notice, exit 0. Hidden test flag `--strict-errors` propagates real exit codes for tests (DESIGN §3.3).
- Timing: emit total elapsed to stderr only under `UNDO_DEBUG=1` (undocumented debug aid; no log files).

## Detailed Requirements

1. Empty/no-match/all-excluded input → exit 0, zero store writes, zero notices (except verbose level).
2. Store unavailable (unwritable root) → warn `store unavailable`, exit 0 (test with chmod 500 store).
3. stdin protocol is the only input path for command text (assert argv contains no command text — code review + test that `ps`-visible args never include a marker string passed via stdin).
4. Integration tests (assert_cmd, real store in tempdir): `rm x` end-to-end produces committed op with correct manifest; `rm a; chmod 600 a` produces ONE op with one `Delete` entry (merge rule); `sudo rm x` produces notice + no op; `rm $X` partial notice + no entries → no op; oversized stdin → warn + exit 0; panic injection via `--strict-errors` off → exit 0.
5. Wall-clock budget: end-to-end small-file case < 150 ms on CI (loose assert < 500 ms).

## Acceptance Criteria

- [ ] All integration scenarios above pass; every `__hook` invocation without `--strict-errors` exits 0 across the entire test suite (enforced by a harness wrapper).
- [ ] Merge behavior verified (one op per line, strength dedup).
- [ ] Version-mismatch invocation emits the warn notice and still snapshots + exits 0.

## Validation

`cargo test hook_entry` green; manual smoke: `echo 'rm /tmp/nope' | target/debug/undo __hook --cwd /tmp --session 01JZK3QG4Y9Z8XWVUTSRQPNMKJ --hook-version 1; echo $?` → 0.

## Dependencies

10, 14, 15, 16, 17, 18.

## Non-goals

The zsh script itself (20); prefilter (20); enable/disable persistence (22).

## Design References

DESIGN.md §5.2, §5.4, §6.4, §7.1, §12.3; ADR-005.
