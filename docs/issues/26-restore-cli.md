# Title

Restore CLI UX (bare `undo`, `undo <id>`)

## Summary

Wire target selection, confirmation prompts, dry-run rendering, and flag handling (`-y --force --dry-run --into`) around the planner (24) and executor (25), matching the DESIGN §3.5 example UX.

## Context

This is the product's headline interaction. Behavior must match §9.2 selection rules and §3.3 exit codes exactly; prompts must be safe non-interactively (fail closed to "no" without `--yes`).

## Scope

- Bare `undo`: select newest op with `kind ∈ {command, restore}`, skipping corrupt and aborted-empty ops (§9.2); none found → `nothing to undo` info, exit 0.
- `undo <OP_ID>`: resolve via 06 (`resolve_op_id`), including prefixes.
- Flow: load+validate manifest (corrupt → exit 5) → plan (24) → conflicts && !--force → print conflict list (escaped, §12.5) + hint `re-run with --force to replace current state (current state will be snapshotted first)` → exit 3.
- `--dry-run`: print the action table (what would be restored where, sizes, replaces-current markers) and exit 0 — no inverse op, no writes.
- Confirmation (TTY on stdin+stderr, no `--yes`): render summary per §3.5 (op id short, command escaped, age, entry count, size) + `Proceed? [y/N]` — default No; non-TTY without `--yes` → exit 2 `confirmation required: pass -y`. `--yes` skips.
- Execute (25); render per-entry results (ok lines terse; failures detailed); final line `undo: restored X/Y entries`; exit per §3.3 (0/4).
- `--into DIR`: DIR must exist and be a directory (else exit 2); no confirmation needed when no current path is replaced (still confirm if TTY? — NO: `--into` never destroys → skip prompt, print destination summary).

## Detailed Requirements

1. Prompt reads a single line from stdin; anything but `y`/`Y` (trimmed) = no → exit 0 with `aborted`.
2. All human output through report/escaping (03); `--json` NOT offered on restore in v1 (explicitly reject with exit 2 `--json not supported for restore`) — automation should use list/show + explicit ids; add a design-reference comment.
3. Integration tests (PTY not required: use `--yes`/non-TTY paths + a PTY-marked `#[ignore]` manual test for the prompt): bare-undo selection skipping a corrupt newest op; prefix restore; conflict exit-3 path with exact hint string; dry-run golden; `--into` on a delete op; non-TTY refusal; nothing-to-undo case on empty store.
4. `undo <id>` where id resolves to an aborted-empty op → exit 2 `operation has no restorable entries`.

## Acceptance Criteria

- [ ] All integration scenarios green; exit codes exactly per §3.3 (0/2/3/4/5 paths each tested).
- [ ] Golden dry-run and conflict outputs (normalized) committed.
- [ ] §3.5 example session reproduced end-to-end in a scripted test (modulo ids/ages).

## Validation

`cargo test cmd_restore` green.

## Dependencies

25 (and transitively 24, 23 not required).

## Non-goals

Interactive picker UI (v2), partial-entry selection flags (v2), JSON restore output (v2).

## Design References

DESIGN.md §3.2–§3.5, §9.2–§9.4, §12.5.
