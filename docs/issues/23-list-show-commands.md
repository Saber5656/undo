# Title

`undo list` and `undo show`

## Summary

Implement the operation-browsing surfaces: `list` (recent ops table via index with scan fallback) and `show <op-id>` (full manifest rendering), both with stable `--json`.

## Context

DESIGN §3.2/§3.5 show the intended output; §8.3 defines the index-as-cache reading rules; §12.5 requires escaping and op-id validation (via 06's `resolve_op_id`).

## Scope

- `undo list [-n N] [--json]` (default N=20): newest first; columns `ID(short 8-char prefix) AGE COMMAND(escaped, truncated 60 cols) ENTRIES SIZE(human)`; restore-kind ops annotated `(inverse of <short-id>)`; corrupt manifests listed as `!! corrupt` rows (id + reason) without failing the command; `partial`/`aborted` ops flagged `~partial`.
- Reading: `index.jsonl` (tolerant reader from 06); when index missing/unreadable or references missing dirs → transparent full scan (no error). `-n 0` = all.
- `undo show <OP_ID> [--json]`: resolve via `resolve_op_id` (prefix support); print header (id, created_at local-time rendered, command, cwd, session, kind/restores, partial/aborted, logical bytes) + entry table (`# DISPOSITION KIND PATH SIZE reason?`) + notices.
- `--json`: list → `{"format":1,"ops":[IndexLine…]}`; show → the manifest verbatim plus `"payload_root"` absolute path.
- Age rendering: `5s`, `3m`, `2h`, `4d` (single coarse unit).

## Detailed Requirements

1. All human-output paths/commands go through `escape_for_terminal` (03).
2. `list` must complete < 100 ms on a 500-op fixture store (loose CI assert < 500 ms) — index path, no manifest reads (§14).
3. `show` on a corrupt manifest → exit 5 with the validation reason (§8.2).
4. Ambiguous/invalid op-id → exit 2 with candidates (06 helper semantics).
5. Tests: golden human output (normalized ids/ages) for a 4-op fixture (command, restore, partial, corrupt); JSON goldens; 500-op timing; prefix resolution paths.

## Acceptance Criteria

- [ ] Fixture goldens pass (human + JSON for both commands).
- [ ] Corrupt op visible in list, refused in show with exit 5.
- [ ] Timing assertion green in CI.

## Validation

`cargo test cmd_list cmd_show` green.

## Dependencies

06.

## Non-goals

Restore selection UX (26); TUI (v2); filtering/search flags (v2).

## Design References

DESIGN.md §3.2, §3.4, §3.5, §8.2–8.3, §12.5, §14.
