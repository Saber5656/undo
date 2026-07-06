# Title

Operation model, manifest schema, index cache

## Summary

Implement the operation data model: manifest (de)serialization exactly matching DESIGN §8.2 format 1, strict read-validation, the staging→rename commit primitive, ULID id generation, and the `index.jsonl` append/read/rebuild cache.

## Context

The manifest is the authoritative record consumed by list/show (23), restore (24/25), GC (27), purge (28); its validation rules are security requirements (DESIGN §12.5).

## Scope

- `src/store/op.rs`: `Manifest`, `Entry`, `Disposition`, `EntryKind`, `OpKind` types with serde field names exactly as in DESIGN §8.2 (jsonc example is normative; `format: 1`).
- ULID generation (crate `ulid`), monotonic within process not required; id == directory name.
- Commit primitive: `StagingOp::create(store, id)` → returns staging dir handle; `StagingOp::commit(manifest)` writes `manifest.json` (serde_json pretty), fsyncs file + staging dir, renames to final, then best-effort `index.jsonl` append under short EX lock (per §8.4; append failure → warning only).
- Read path: `Operation::load(store, id) -> Result<Manifest>` with strict validation: format known; id matches dir and ULID alphabet `[0-9A-HJKMNP-TV-Z]{26}`; every `path`/`move_dest` absolute, no `..` component, no empty components; `payload` matches `^payload/[0-9]+$`; `entries[].index` unique. Violation → `StoreIntegrity` error carrying the reason.
- Index: `IndexLine` struct per §8.3; `Index::append(line)`, `Index::read_all()` tolerating trailing garbage/partial last line; `Index::rebuild(store)` by scanning committed op dirs (used by 27; implement here, wire there).
- Op-id prefix resolution helper: `resolve_op_id(store, input) -> Result<Ulid>` accepting full id or unique prefix ≥ 4 chars after charset validation; ambiguous → error listing up to 5 candidates.

## Detailed Requirements

1. Serialization round-trip must be byte-stable for a fixed input (test with a golden file `tests/fixtures/manifest-v1.json`).
2. Unknown manifest fields are ignored on read (forward compatibility within major format).
3. `created_at` RFC3339 UTC with millisecond precision.
4. Validation rejection cases each produce distinct reason strings (tested): relative path, `..`, bad payload ref, id mismatch, bad format.
5. Charset validation of user-supplied op-ids happens **before** any path construction (§12.5) — enforce by API shape: `resolve_op_id` is the only entry point taking raw user input.

## Acceptance Criteria

- [ ] Golden-file round-trip test passes; a manifest edited to `"path": "../x"` fails load with the traversal reason.
- [ ] Commit primitive: crash simulation (kill between payload write and rename — spawn helper process) leaves only `.staging`; committed dirs always contain valid manifest.
- [ ] Index read tolerates a truncated final line; rebuild reproduces index equal to committed ops.
- [ ] Prefix resolution: exact, unique-prefix, ambiguous (two ops sharing prefix), invalid charset (`../../x` → charset error, no fs access attempted — assert via strace-less logic test).

## Validation

`cargo test store::op store::index` green; golden fixture committed.

## Dependencies

04, 05.

## Non-goals

Payload writing (07/08), plan semantics (§6), GC compaction logic (27).

## Design References

DESIGN.md §8.2, §8.3, §8.4, §7.6, §12.5; ADR-004.
