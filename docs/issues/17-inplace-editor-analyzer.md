# Title

sed/perl in-place analyzer

## Summary

Implement the in-place editor analyzer: detect `-i` mode across three grammars (BSD sed, GNU gsed, perl), classify script vs file operands, and plan `Overwrite` entries for the files that will be rewritten.

## Context

DESIGN §6.7 and research/macos-platform-notes.md §5: BSD `sed -i` takes a *separate* suffix argument while GNU attaches it — the single most error-prone grammar in the product. Mis-parsing here means either missed protection or snapshotting a script file; the design resolves ambiguity toward snapshotting more (never mis-consuming operands).

## Scope

- `src/analyze/cmd/inplace.rs` (shared trait/harness), grammar keyed by head name:
  - `sed` (BSD): `-i` → next word is suffix (possibly empty `''` → empty unquoted_text) and is consumed; attached `-iSUFFIX` also accepted. Valued flags `-e -f -i`; booleans `-a -E -g -H -h -l -n -r -s -u -x`? — implement exactly the macOS man page set: `-a -E -G -g -H -h -i -l -n -r -s -u -x` with `-e/-f/-i` valued.
  - `gsed` (GNU): `-i[SUFFIX]` attached-only (separate next word is a FILE); `--in-place[=SUFFIX]`; valued `-e/--expression`, `-f/--file`; booleans per GNU man (`-n -r -E -s -u -z --posix …` — unknown long flags without `=` treated boolean, with `=` treated self-contained).
  - `perl`: in-place iff `-i[ext]` attached or inside a cluster (`-pi`, `-pi.bak`, `-i.bak`); valued `-e/-E CODE` (repeatable); cluster splitting for single-dash groups (`-pie` → `-p -i -e` — note: real perl treats trailing `e` in cluster as taking the NEXT arg as code; implement exactly that); if no `-e/-E`, first non-flag operand is the program file → excluded from plan; remaining operands are files.
- No `-i` detected → empty result (no op), regardless of file operands.
- Each in-place file operand surviving resolution → `Overwrite/Content` (existing files only).
- Backup-suffix nuance: when a non-empty suffix is given (sed `-i.bak` / perl `-i.bak`), the editor itself keeps a backup — still snapshot (undo history + uniform behavior); verbose notice `editor keeps its own backup (.bak)`.

## Detailed Requirements

1. Grammar table tests ≥ 35 cases, including at minimum:
   - `sed -i '' 's/a/b/' f.txt` (BSD classic) → f.txt planned;
   - `sed -i.bak 's/a/b/' f` → f planned + verbose backup notice;
   - `sed -i 's/a/b/' f` (BSD trap: suffix consumes the script!) → per BSD grammar `'s/a/b/'` is the suffix, `f` is the script, **no file operands** → empty plan + warn-level notice `ambiguous sed -i usage` (explicit requirement: detect this common-mistake shape — one remaining operand after suffix consumption — and emit the ambiguity notice instead of silently planning nothing);
   - `sed 's/a/b/' f` (no -i) → empty;
   - `gsed -i 's/a/b/' f` → f planned;
   - `gsed -i.bak -e 's/x/y/' a b` → a,b planned;
   - `sed -i '' -e one -e two f g` → f,g;
   - `perl -pi -e 's/a/b/' f` → f; `perl -i.bak -pe …` → f; `perl -pie 's/…/…/' f` (cluster-trailing-e) → code consumed, f planned; `perl script.pl f` (no -i) → empty; `perl -i script.pl f` (no -e: script.pl excluded) → f planned;
   - globbed files `sed -i '' expr *.conf`; unresolvable `$F`; `--` usage.
2. Every ambiguity fallback documented in code comments referencing §6.7's "snapshot more rather than less" rule and never consuming past `--`.
3. U5 probe: verify perl cluster behavior against `man perlrun` and record findings in the PR description.

## Acceptance Criteria

- [ ] All grammar table cases pass exactly.
- [ ] BSD `sed -i 's/…/…/' f` mistake-shape emits the ambiguity warn notice.
- [ ] No plan ever includes a word consumed as script/suffix/expression.

## Validation

`cargo test analyze::cmd::inplace` green; case table committed as data alongside tests.

## Dependencies

13, 14 (harness).

## Non-goals

Other editors (`ex`, `ed`, `sd` — v2 candidates); running sed/perl.

## Design References

DESIGN.md §6.7; research/macos-platform-notes.md §5; §17 U5.
