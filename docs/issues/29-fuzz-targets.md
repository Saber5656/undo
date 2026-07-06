# Title

Fuzz targets for parser boundary + CI smoke

## Summary

Add cargo-fuzz targets for the lexer and the full analysis pipeline (lex→extract→resolve→analyzers in no-filesystem mode), seed corpora, and a time-boxed CI smoke job.

## Context

The parser consumes untrusted command text (DESIGN §12.1); "never panics, never evaluates" is a security acceptance criterion that fuzzing enforces mechanically (ADR-001, §12.9).

## Scope

- `fuzz/` workspace (cargo-fuzz, libFuzzer): targets:
  1. `lex_raw`: arbitrary bytes → lossy string → `lex()` (11's exported entry) — asserts no panic;
  2. `pipeline_nofs`: arbitrary string → lex → extract → analyzers with a **stubbed resolver** (no filesystem access: existence oracle answering pseudo-randomly from a hash of the path; glob expansion replaced by identity) — asserts no panic and invariant checks: no plan entry with relative path, no entry path containing `..` component, plans empty for non-matched heads.
- Seed corpus: every table-test input from issues 11–18 exported into `fuzz/corpus/` via a small generator test (`cargo test --features corpus-export` writes them).
- CI job (extends 02's workflow): nightly-toolchain fuzz smoke `cargo +nightly fuzz run <target> -- -max_total_time=60` per target, on `macos-15`, non-blocking-on-main? — blocking: run on PRs touching `src/analyze/**` or `fuzz/**` (path filter), always on a weekly schedule (`schedule:` cron) against `main`; failures upload the crashing input as artifact.
- `fuzz/README.md`: how to reproduce a crash locally.

## Detailed Requirements

1. The no-fs guarantee of `pipeline_nofs` must hold structurally: the fuzz target compiles the resolver against a trait; the fs-backed impl lives behind the trait too (small refactor of 13 acceptable — coordinate; the trait likely already exists for tests).
2. Both targets must reach > 1,000 execs/s locally (sanity: no accidental I/O in the loop) — record numbers in PR.
3. Any crash found during development gets a regression unit test in the owning module before this issue closes.
4. Corpus checked in (small, text) — cap 200 files.

## Acceptance Criteria

- [ ] `cargo +nightly fuzz run lex_raw -- -max_total_time=60` and `pipeline_nofs` complete crash-free locally and in the CI smoke job.
- [ ] Invariant assertions active in `pipeline_nofs` (not just no-panic).
- [ ] Weekly schedule + path-filtered PR trigger visible in workflow file; artifact upload on failure configured.

## Validation

CI run link in PR; local exec/s numbers recorded.

## Dependencies

11, 12, 13, 14, 15, 16, 17, 18 (targets exist).

## Non-goals

Fuzzing store/manifest parsing (JSON via serde — lower risk; v2 candidate), differential fuzzing vs real zsh (v2 idea).

## Design References

DESIGN.md §12.1, §12.9, §15; ADR-001.
