# Title

`undo purge`

## Summary

Implement immediate manual disposal: `undo purge <OP_ID>` and `undo purge --all`, with confirmation, EX locking, and honest deletion semantics.

## Context

Purge is the user's data-lifetime control for sensitive snapshots (ADR-006). It shares eviction machinery with GC (27) but is user-directed and bypasses retention guards (including min-keep).

## Scope

- `undo purge <OP_ID> [--yes]`: resolve id (06), TTY confirm (`This permanently deletes the snapshot for: <command>. Proceed? [y/N]`) unless `--yes`; non-TTY without `--yes` → exit 2 (same rule as 26); EX lock; evict via 27's rename-to-trash + remove; compact index line out.
- `undo purge --all [--yes]`: confirmation states op count + total logical bytes; removes every op (incl. staging/trash residue); store root and config remain.
- Output: `undo: purged <n> operation(s), <bytes> reclaimed (logical)` + the honesty line `note: this deletes data but does not shred underlying blocks (APFS/SSD)` — exact string, README/SECURITY.md reference it (ADR-006).
- Purging an op referenced by a restore-op's `restores` field is allowed (dangling reference tolerated; `show` renders `restores: 01… (purged)` — add tolerant rendering to 23 via a small follow-up edit within this issue).

## Detailed Requirements

1. Purge ignores `min_keep_minutes` and the newest-op guard (user intent overrides).
2. Contended EX lock (restore in progress) → bounded wait (05's 2 s) then exit 1 `store busy (restore or gc in progress)`.
3. Tests: single purge (dir gone, index compacted, list no longer shows it); `--all` on 5-op fixture; confirmation refusal path (`n` → exit 0, nothing deleted); non-TTY refusal; busy-lock path; dangling `restores` rendering.

## Acceptance Criteria

- [ ] All test paths green; honesty note present in output (golden).
- [ ] After `purge --all`: `undo list` empty, `undo status` shows zero usage, store still healthy (`doctor` ok).

## Validation

`cargo test cmd_purge` green.

## Dependencies

27 (eviction machinery), 06.

## Non-goals

Secure shredding (impossible to guarantee on APFS/SSD — documented instead), selective per-entry purge (v2).

## Design References

DESIGN.md §10.4, §3.2; ADR-006.
