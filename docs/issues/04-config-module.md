# Title

Config module and `undo config` command

## Summary

Implement TOML configuration loading with the complete v1 key set, defaults, precedence (CLI > env > file > default), validation, and the `undo config show|path` surface.

## Context

DESIGN §11 defines the exact schema. Limits/retention/exclusion values drive the snapshot engine and GC; `[protect]` toggles drive analyzers; `ui.notices` drives issue 03's sink.

## Scope

- `src/config.rs`: `Config` struct mirroring DESIGN §11 exactly (serde + toml), `Config::default()` with the documented default values, `Config::load(cli_overrides) -> (Config, Vec<Warning>)`.
- Path resolution: `--config` flag > `UNDO_CONFIG` > `${XDG_CONFIG_HOME:-~/.config}/undo/config.toml`; missing file → defaults silently.
- Byte-size parser: integer bytes or string with `KiB|MiB|GiB` suffix (case-sensitive); invalid → load error naming the key.
- `exclude.paths`: tilde-expand at load; store root is appended unconditionally at runtime (not configurable).
- Unknown keys produce warnings (serde `deny_unknown_fields` OFF; collect unknowns via `toml::Value` pre-pass), not errors.
- File size guard: config > 1 MiB → refuse (`StoreIntegrity`-class message, exit 5? No — config error is `Runtime`, exit 1; store codes are store-only).
- Top-level `enabled` bool (default true) — written by issue 22.
- `undo config path` prints the resolved path (even if missing); `undo config show` prints effective TOML (defaults merged, secrets-free by construction) to stdout.

## Detailed Requirements

1. Every DESIGN §11 key present with the exact default: protect.* all true; limits 512MiB/50000/2000; retention 7/500/2GiB/10; exclude defaults `["~/Library/Caches", "~/.Trash"]`; ui warn/auto.
2. Validation rules: `max_entries_per_op ≥ 100`; `snapshot_soft_deadline_ms ≥ 100`; `retention.max_ops ≥ 1`; `min_keep_minutes ≥ 0`; notices/color enum values — violations are load errors with key-named messages.
3. Precedence covered by tests: env beats file, flag beats env (use `--store`/`UNDO_STORE` analog only for config path here).
4. `config show` output must be valid TOML re-parseable into an equal `Config` (round-trip test).

## Acceptance Criteria

- [ ] Unit tests: defaults, precedence, byte-size parsing (`"512MiB"`, `"2GiB"`, `1024`, invalid `"2GB"` fails), unknown-key warning, oversize file refusal, validation failures.
- [ ] `undo config show` on a machine with no config prints the full default set; exit 0.
- [ ] `undo config path` prints the XDG-resolved path; honors `UNDO_CONFIG`.

## Validation

`cargo test config::` green; manual: `UNDO_CONFIG=/tmp/x.toml target/debug/undo config path`.

## Dependencies

03.

## Non-goals

Store creation (05); `enable/disable` writing logic (22); watching for config changes.

## Design References

DESIGN.md §11, §3.2; ADR-006 (exclusion rationale).
