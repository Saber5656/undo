# Title

Word resolution: tilde, globs, absolutization

## Summary

Resolve operand words to absolute filesystem paths: tilde expansion, zsh-default-compatible glob expansion via the `glob` crate, lexical absolutization, existence filtering, and exclusion-list application — never evaluating shell constructs.

## Context

DESIGN §6.3 defines the pipeline; §6.4 defines partial/not-protected outcomes. Fidelity target: match zsh *default* semantics (no dotfile globbing, case-sensitive) — documented divergences are acceptable, silent wrong expansion is not.

## Scope

- `src/analyze/resolve.rs`: `fn resolve_words(words, cwd, globbing_disabled, excludes) -> Resolution { paths: Vec<ResolvedPath>, partial: bool, notices }`.
- Unresolvable words → skipped, `partial = true` (notice `argument not resolvable before execution`, once per command).
- Tilde: leading unquoted `~` → `$HOME`; `~name` → `getpwnam`; unknown user → unresolvable-word treatment.
- Glob (only when `has_glob` && !globbing_disabled && the metas were unquoted): `glob::glob_with` anchored at `cwd` for relative patterns, options: `require_literal_leading_dot: true`, `require_literal_separator: true`, `case_sensitive: true`; results sorted; zero matches → contributes nothing (per §6.3; zsh NOMATCH nuance documented in a code comment referencing F6).
- Absolutization: join `cwd`, lexically normalize `.`/`..`; do not canonicalize symlinks (lstat semantics preserved downstream).
- Existence: `lstat`; missing → dropped (verbose notice).
- Exclusion: store root always + config `exclude.paths` prefix match on the absolute path → dropped (verbose notice).

## Detailed Requirements

1. `**/` recursive glob supported (glob crate) — test parity note: zsh `**/*.log` vs crate behavior compared in tests for the fixture tree; any divergence documented in code comments and README limitation list (issue 32 picks it up from here).
2. Words that had quotes around glob metas (`'*.log'`) are literal — no expansion (lexer's per-segment info; conservative rule: if any part of the word was quoted and contains metas, treat whole word literal).
3. Path with trailing slash preserved semantically (points at dir; lstat works the same).
4. Table tests ≥ 30 cases: `~`, `~root`, `~nosuchuser`, `*.log` (with/without dotfiles present), `.*` explicit dotfile pattern, `**/x`, `'*.log'` quoted, `noglob` command, `../sibling`, absolute operand, missing path, excluded path (under `~/Library/Caches`), store-path operand, `$HOME/x` (unresolvable), empty-after-resolution command.
5. No syscalls other than `lstat`/`getpwnam`/glob's readdir — no writes (grep gate as in 11).

## Acceptance Criteria

- [ ] Table cases pass; dotfile invariant: `*` never matches `.hidden` fixture; `.*` does.
- [ ] Store-path operand is dropped with verbose notice (self-reference guard first line — §12.4).
- [ ] Unresolvable-word command yields `partial: true` + exactly one warn notice.

## Validation

`cargo test analyze::resolve` green; parity comparison test output attached to PR.

## Dependencies

04 (excludes), 12.

## Non-goals

Command-specific operand classification (14–18); any parameter/command substitution (never).

## Design References

DESIGN.md §6.3, §6.4, §12.4; ADR-001; §13 F6.
