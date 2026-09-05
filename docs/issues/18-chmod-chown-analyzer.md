# Title

chmod/chown analyzer

## Summary

Implement the metadata-command analyzer: mode/owner spec detection, path operand extraction, `-R` recursive planning with caps, producing `Metadata` entries (no payloads).

## Context

DESIGN §6.8. Restore only needs *prior* attributes (captured by issue 09); the analyzer just decides which paths and whether recursion applies.

## Scope

- `src/analyze/cmd/meta.rs` (shared trait/harness) for heads `chmod|gchmod|chown|gchown`.
- Grammar per §6.8: booleans `-f -h -v -R -H -L -P`; first non-flag operand = mode/owner spec (not validated); remaining operands = paths; `--` respected; `gchmod/gchown --reference=RFILE` → spec absent, mark command `partial` (reference file NOT an operand), paths still planned.
- Entries: each existing path → `Metadata/MetadataOnly`; `recursive: true` carried on the entry when `-R` (orchestrator/09 walk uses it).
- Symbolic (`u+x`), octal (`0644`), and owner (`user:group`, `:group`) specs all treated opaquely (one test each proving they're consumed as spec, not path).

## Detailed Requirements

1. Table tests ≥ 20 cases: `chmod 644 f`; `chmod -R 755 dir`; `chmod u+x a b`; `chmod -h 644 link` (entry = link itself); `chown alice f`; `chown alice:staff -R dir` (GNU-style flag-after-spec ordering: BSD requires flags first — accept flags in any position conservatively, document); `chown :staff f`; `gchmod --reference=r f` (partial, f planned); `chmod 644 $F` (partial); `chmod 644 *.sh`; `chmod -- 644 -f` (dash-named file); missing path (dropped); spec-only (no paths → empty).
2. The `recursive` flag must NOT trigger a walk here (that's snapshot-time, 09/10) — analyzer stays cheap.
3. Cross-command merging (`rm a; chmod 644 a` → one op, `Delete` wins) happens at the pipeline level per DESIGN §7.1 — implemented in issue 19, NOT here. This analyzer returns per-command results only; add a code comment pointing at §7.1.

## Acceptance Criteria

- [ ] All table cases pass with exact entry sets and `recursive` flags.
- [ ] `--reference` case yields partial + planned paths.
- [ ] Spec words never appear as planned paths.

## Validation

`cargo test analyze::cmd::meta` green.

## Dependencies

13, 14 (harness).

## Non-goals

Attribute capture (09), applying attributes (25), `chflags` (v2).

## Design References

DESIGN.md §6.8, §7.5; §13 F5.
