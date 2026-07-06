# Title

enable / disable / status commands

## Summary

Implement the persistent on/off switch (`undo enable|disable` writing the config `enabled` key) and `undo status` (hook state, store statistics, retention usage, version).

## Context

DESIGN §3.2; the session-scoped `UNDO_DISABLE=1` env path already exists in the hook (20); this issue adds the persistent path (`__hook` already gates on config `enabled`, 19) and the visibility surface.

## Scope

- `undo disable`: set top-level `enabled = false` in the config file (create file/dirs if missing, preserving existing content and comments is NOT required — rewrite via toml round-trip of the parsed doc; user comments may be lost: print a notice when the file previously existed `note: config rewritten; comments not preserved`).
- `undo enable`: set `enabled = true` (or remove the key).
- Both print the new state and remind about `UNDO_DISABLE` for per-session control.
- `undo status` output (human): enabled (config) / this-shell hook active (`UNDO_SESSION`) / binary version + hook version constant / store path / ops count / logical bytes vs `retention.max_total_bytes` / oldest op age vs `max_age_days` / last op summary (id, age, command escaped). `--json` with `format:1` and fixed keys.
- Status reads via index (06) with scan fallback; missing store → zeros + `store not yet created`.

## Detailed Requirements

1. Config write is atomic (temp + rename in config dir).
2. `disable` while config file has parse errors → refuse with the parse error (don't clobber a broken file silently).
3. Status must not take locks (read-only, tolerate concurrent writers; §8.4).
4. Tests: enable→disable→enable round-trip preserves other keys (`retention.max_ops = 7` fixture survives); status golden JSON on a fixture store with 3 ops; broken-config refusal.

## Acceptance Criteria

- [ ] `undo disable` → subsequent `__hook` invocation is a silent no-op (integration test chaining 19).
- [ ] Round-trip preserves unrelated keys; broken config refused.
- [ ] `status --json` golden test green; human output shows retention usage percentages.

## Validation

`cargo test cmd_toggle status` green.

## Dependencies

04, 06, 20 (semantics), 19 (`enabled` gate exists).

## Non-goals

Per-command toggles (config `[protect]` is issue 04); uninstall command.

## Design References

DESIGN.md §3.2, §5.3, §11.
