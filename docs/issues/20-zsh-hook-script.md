# Title

zsh hook script and `undo init zsh`

## Summary

Author the static zsh integration script (preexec hook with fast prefilter, fail-open spawning of `__hook`) embedded in the binary and emitted by `undo init zsh`; verify preexec behavior under CI (`zsh -ic`) including the U1/U3 known unknowns.

## Context

DESIGN §5.1 defines hook responsibilities; §12.2 requires the script be a compile-time constant (no interpolation); §14 sets the prefilter budget. This issue also owns the empirical probes: does preexec fire under `zsh -ic` (U1), and does it fire before NOMATCH abort (U3)?

## Scope

- `assets/hook.zsh` embedded via `include_str!`; `undo init zsh` prints it verbatim to stdout (exit 0). First line comment: `# undo hook v1 (do not edit; managed by 'undo init zsh')`.
- Script per DESIGN §5.1 exactly:
  1. interactive guard; idempotence guard (`__UNDO_LOADED`);
  2. `command -v undo` check → one-time warning + self-disable when missing;
  3. `UNDO_SESSION` ULID export if unset (generate via `undo __session-id` — add this trivial hidden subcommand printing a fresh ULID; zsh cannot make ULIDs);
  4. `autoload -Uz add-zsh-hook && add-zsh-hook preexec __undo_preexec`;
  5. `__undo_preexec`: `UNDO_DISABLE` check; `local cmd=${3:-$1}`; prefilter; on match `print -r -- "$cmd" | command undo __hook --cwd "$PWD" --session "$UNDO_SESSION" --hook-version 1 || true`.
- Prefilter (§5.1): single zsh `[[ $cmd =~ … ]]` (or `${(M)…}` pattern) matching protected head tokens `rm mv cp sed perl chmod chown grm gmv gcp gsed gchmod gchown` at line start, after `; && || | ( { ` sequences, or after a `/` (path-invoked), as whole words. Must be a **superset** matcher (false positives OK). Keep it one static pattern string with a comment table.
- All hook-internal names prefixed `__undo_` / `__UNDO_` (§5.4 namespace rule); no `setopt` changes; `$?` preserved (prefilter runs in a function; ensure no command alters user-visible state — end function with `return 0`? preexec return value is ignored by zsh; still return 0 explicitly).
- Shell-level tests (`tests/shell_hook.rs` spawning real `/bin/zsh`):
  - U1 probe: `zsh -ic 'rm <tmpfile>'` with hook loaded via `ZDOTDIR` fixture `.zshrc` → op committed? Record result; if preexec doesn't fire without PTY, implement the `zpty`-based fallback harness (documented in DESIGN §15) and use it for all L2 tests.
  - interception: protected command creates op; non-protected (`ls`) spawns nothing (assert via absence of ops + `UNDO_DEBUG` marker technique);
  - fail-open matrix: binary renamed away after init → command still runs; store chmod 500 → command still runs + warn; `UNDO_DISABLE=1` → no op;
  - `$?` preservation: `false; <protected cmd>` sequence — user-visible `$?` semantics unchanged (script test);
  - U3 probe: `rm *.nomatch` in empty dir — record whether an op/notice appears; assert only "no crash, shell healthy".

## Detailed Requirements

1. Script must pass `zsh -n` (syntax check) in CI.
2. No string from config/env is spliced into emitted code (§12.2) — the script is byte-identical across machines (test: run `init` twice under different HOME/env → identical output).
3. Prefilter benchmark: a zsh loop of 10,000 non-matching preexec calls completes < 5 s in CI (≈ <0.5 ms each; document methodology in the test).
4. `undo init bash` etc. → exit 2 usage error `only zsh is supported in v1`.
5. Findings for U1/U3 must be written into the PR description AND, if behavior differs from DESIGN assumptions, DESIGN §17/§13 updated in the same PR.

## Acceptance Criteria

- [ ] `eval "$(undo init zsh)"` in a fresh interactive zsh: `rm` of a temp file is intercepted (op exists) and the file is actually removed (command unmodified).
- [ ] Fail-open matrix green; `ls`-class commands spawn no `undo` process.
- [ ] `zsh -n` clean; init output byte-stable; prefilter benchmark passes.
- [ ] U1/U3 findings documented; L2 harness (direct `-ic` or zpty fallback) reusable by issue 31.

## Validation

`cargo test shell_hook` green on `macos-15` + `macos-26` CI.

## Dependencies

19.

## Non-goals

.zshrc auto-editing (never — user adds the eval line per README); bash/fish; completions.

## Design References

DESIGN.md §5.1–§5.4, §12.2, §14, §17 U1/U3; ADR-001; ADR-005.
