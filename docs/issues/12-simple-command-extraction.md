# Title

Simple-command extraction and head resolution

## Summary

Split the token stream into simple commands, strip prefixes (env assignments, wrappers, `noglob`, `env`), detect `sudo`/`xargs` policies, resolve command heads to the protected-command table, and emit the §6.4 notices for policy skips.

## Context

DESIGN §6.2 defines the exact prefix and policy rules; the protected-command table maps heads to analyzers (14–18). This stage decides *whether* a command is analyzed at all.

## Scope

- `src/analyze/extract.rs`: `fn extract(tokens) -> Vec<SimpleCommand>`; `SimpleCommand { head: HeadMatch, args: Vec<WordToken>, globbing_disabled: bool, policy: Policy }`.
- Splitting at `Semi|AndIf|OrIf|Pipe|PipeAmp|Amp|LParen|RParen|LBrace|RBrace|Newline`; redirect ops and their target words removed from args (target = next word token); heredoc op → remaining command marked `Policy::Unsupported("heredoc")`.
- Prefix stripping loop per DESIGN §6.2: `NAME=value` words (POSIX name regex `[A-Za-z_][A-Za-z0-9_]*=`); wrappers `command builtin exec nocorrect time`; `noglob` → `globbing_disabled = true`; `env` → strip its `-i`, `-u NAME` and `NAME=value` args.
- `sudo`/`doas` head → `Policy::Sudo` (stop); `xargs` head → `Policy::StdinDriven` (stop).
- Head resolution: basename if word contains `/`; unresolvable head word → no match; lookup in the table (DESIGN §6.2) filtered by `[protect]` config toggles (family off → treated as unmatched, verbose notice).
- Notice emission (via report, exact §6.4 strings): sudo → `sudo commands are out of scope`; xargs → `stdin-driven arguments`; once per input line per reason.

## Detailed Requirements

1. Pipeline members each become their own `SimpleCommand` (`foo | xargs rm` → foo unmatched; xargs policy note).
2. Env-assignment stripping must not consume `a=b` when it is an *operand* (only leading positions before the head).
3. Table-driven tests ≥ 40 cases: `rm x`; `\rm x` (lexer gives word `rm` — backslash escape); `command rm -rf x`; `/bin/rm x`; `sudo rm x`; `env -i FOO=1 rm x`; `noglob rm *`; `VAR=1 rm x`; `rm x; mv a b`; `echo hi && rm x`; `find . | xargs rm`; `time rm x`; `rm x > /tmp/log` (redirect target excluded); heredoc case; `grm x`; protect.rm=false config case; unresolvable head `$CMD x`.
4. Output preserves original word order of args (analyzers depend on it).

## Acceptance Criteria

- [ ] All table cases produce the expected `SimpleCommand` summaries (head match, args, policy, flags).
- [ ] Notices asserted: sudo and xargs cases emit exactly one warn-level notice each with the normative strings.
- [ ] `[protect] rm = false` config drops rm from matching without affecting mv.

## Validation

`cargo test analyze::extract` green.

## Dependencies

11.

## Non-goals

Word expansion (13), per-command flag grammars (14–18).

## Design References

DESIGN.md §6.2, §6.4; ADR-001 (sudo boundary).
