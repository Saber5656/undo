# Title

GC, retention enforcement, index self-heal

## Summary

Implement retention policy enforcement: `undo gc` (with `--dry-run`), staging/trash sweeping, eviction ordering with the min-keep and newest-op guards, index compaction/rebuild, and the post-snapshot auto-GC trigger.

## Context

DESIGN §10 defines policy and triggers; §8.3–8.4 define index self-heal and the EX-lock discipline; ADR-006 makes bounded data lifetime a security property.

## Scope

- `src/lifecycle.rs`:
  - `fn gc(store, config, dry_run) -> GcReport` under EX lock (05): sweep `.staging`/`.trash` older than 1 h; evaluate committed ops oldest-first against `max_age_days` → `max_ops` → `max_total_bytes` (logical bytes from index, rebuilding first if inconsistent); never evict ops younger than `min_keep_minutes`; size-cap eviction never removes the newest op (§10.1); eviction = rename to `<id>.trash` + recursive remove; compact `index.jsonl` to survivors.
  - Index self-heal: `Index::rebuild` (06) triggered when: index missing, unparseable beyond tolerance, references a missing dir, or misses an existing dir (detect via cheap dir-count comparison).
  - `fn auto_gc_check(store)` (called by 10 post-commit): threshold pre-check WITHOUT lock (index length, oldest ULID timestamp vs age cap); if exceeded → `try_lock_exclusive` (non-blocking; contended → skip silently); run gc with a 250 ms wall budget, stopping mid-eviction when exceeded (resume next trigger).
- `GcReport` (and `--dry-run` rendering): per-op eviction reason (`age`/`count`/`size`), bytes reclaimed (logical), staging swept, index rebuilt (bool).

## Detailed Requirements

1. Eviction order strictly ULID-ascending; property test: after gc, the surviving set is exactly the newest ops satisfying all caps, modulo the min-keep guard.
2. min-keep guard rule (DESIGN §10.1 as written): an op younger than `min_keep_minutes` is **never** evicted by age/count/size enforcement; the count/size caps tolerate temporary overshoot until the ops age past the guard. Add a code comment noting that runaway-loop overshoot is bounded by the next gc after the guard window.
3. Crash-safety: kill -9 during eviction (spawned-process test) leaves either the op intact or a `.trash` dir; next gc completes removal; index rebuilt consistent.
4. Auto-GC budget: instrument and assert < 400 ms on a fixture needing 50 evictions (loose CI bound).
5. Tests: age eviction (backdated ULIDs — generate ids with old timestamps), count eviction, size eviction sparing newest, min-keep survival, staging sweep (fresh staging survives, old swept), dry-run mutates nothing (tree hash before/after), contended-lock skip (hold EX in helper process), index self-heal in all four trigger cases.

## Acceptance Criteria

- [ ] Every retention dimension tested individually + one combined scenario.
- [ ] Kill-during-eviction test converges on next run.
- [ ] `gc --dry-run` output golden; store tree unchanged.
- [ ] Auto-GC after a snapshot evicts on an over-cap fixture store (integration with 10's trigger).

## Validation

`cargo test lifecycle::gc` green on CI.

## Dependencies

06 (index/rebuild), 05 (locks); 10's trigger wiring (stub replaced here).

## Non-goals

purge (28), pinning (v2), physical-bytes accounting (v2).

## Design References

DESIGN.md §10.1–§10.3, §8.3–§8.4, §13 F3/F9; ADR-004; ADR-006.
