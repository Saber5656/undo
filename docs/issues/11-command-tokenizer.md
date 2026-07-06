# Title

Command-line tokenizer (no-eval security boundary)

## Summary

Implement the conservative lexer that turns raw zsh command text into words and operators, flagging unresolvable constructs — without ever executing, expanding, or evaluating anything.

## Context

This is the primary untrusted-input boundary (DESIGN §12.1; ADR-001). Correctness bar: never panic, never mis-tokenize into evaluation; fuzzed in issue 29. All analyzers consume its output.

## Scope

- `src/analyze/lex.rs`: `fn lex(input: &str) -> ParseOutcome` where `ParseOutcome = Tokens(Vec<Token>) | Unsupported(reason)`.
- `Token = Word(WordToken) | Op(OpKind)`; `WordToken { text: String, quoted: bool, has_glob: bool, unresolvable: bool }`; `OpKind = Semi | AndIf | OrIf | Pipe | PipeAmp | Amp | LParen | RParen | LBrace | RBrace | Newline | Redirect(RedirKind)`.
- Lexing rules exactly per DESIGN §6.1:
  - whitespace splitting; `'…'` (no inner escapes); `"…"` with `\"`/`\\`; backslash escapes outside quotes; adjacent quoted/unquoted segments concatenate into one word (`a"b"c`).
  - `$` (unquoted or in double quotes), backtick, `$(`, `${`, `<(`, `>(`, `=(` anywhere in a word → `unresolvable: true` (still a single word; do not attempt to find the construct's end beyond quote correctness).
  - `<<` (heredoc) → `Op(Redirect(Heredoc))`; consumer treats rest of simple command conservatively (12).
  - redirection forms: `<`, `>`, `>>`, `&>`, `>&`, `<<<`, and digit-prefixed (`2>`, `2>>`) as `Redirect` ops; the following word is the redirect target (marked by the extractor, not the lexer).
  - glob detection: unquoted `*`, `?`, `[` present → `has_glob` (quoted metas are literal → `quoted: true` and no glob flag for those segments).
- Limits (DESIGN §12.1): input > 1 MiB, > 65,536 tokens, quote/paren nesting > 64 → `Unsupported` with reason.
- Invalid UTF-8 is handled upstream (lossy conversion at the `__hook` read, issue 19); lexer takes `&str`.

## Detailed Requirements

1. Table-driven tests with ≥ 60 cases covering at minimum: plain words; all operators; each quote form; concatenated segments; every unresolvable trigger; unterminated quote (→ `Unsupported`); trailing backslash; multiline input; UTF-8 (Japanese filename); glob chars quoted vs unquoted; `--`; empty input; whitespace-only.
2. Property invariant asserted in tests: lexer performs no syscalls (pure function; enforce by module review + no `std::fs`/`std::process` imports — add a grep gate to justfile like issue 03's).
3. Never panics: fuzz harness entry `pub fn lex_fuzz_entry(data: &[u8])` exported for issue 29.
4. Word text preserves original characters (no unescaping loss for later display), but a separate `unquoted_text` is provided with quotes/escapes removed for analyzer consumption. Both carried on `WordToken`.

## Acceptance Criteria

- [ ] All table cases pass; a documented case list lives beside the tests as data (rows: input → expected token summary).
- [ ] `lex("rm \"$(x)\" 'a b' c*")` yields: word rm; word unresolvable; word quoted `a b`; word has_glob.
- [ ] `Unsupported` on the three limit violations and unterminated quotes; no panic on any test input.
- [ ] grep gate: no fs/process imports in `analyze/lex.rs`.

## Validation

`cargo test analyze::lex` green; case-table file present.

## Dependencies

03.

## Non-goals

Simple-command semantics (12), expansion (13), zsh-specific syntax beyond DESIGN §6.1 (extended glob qualifiers etc. — unresolvable/conservative).

## Design References

DESIGN.md §6.1, §12.1; ADR-001.
