# Title

Cargo project scaffolding and CLI skeleton

## Summary

Create the Rust project skeleton for the `undo` binary: cargo layout, pinned toolchain, module tree matching the architecture, and a clap-based CLI with all v1 subcommands stubbed.

## Context

Everything else builds on this layout. The module boundaries come from DESIGN.md §4; the CLI surface from §3.2. No behavior is implemented here beyond `--version`, `--help`, and stub exits.

## Scope

- `Cargo.toml` (package `undo`, `edition = "2024"`, `[[bin]] name = "undo"`), `Cargo.lock` committed.
- `rust-toolchain.toml` with `channel = "stable"`.
- `.gitignore` (target/, fuzz artifacts), `rustfmt.toml` (defaults; explicit file), `clippy` config only if needed.
- `src/main.rs` + module files: `cli.rs`, `config.rs`, `report.rs`, `store/mod.rs`, `analyze/mod.rs`, `snapshot/mod.rs`, `restore/mod.rs`, `lifecycle.rs`, `doctor.rs` — each containing a doc comment naming its DESIGN section and `// TODO(issue-NN)` markers.
- clap v4 derive CLI implementing the grammar of DESIGN §3.2 exactly, including hidden `__hook` and global flags (`--store`, `--config`, `--json`, `-y/--yes`, `--force`, `--dry-run`, `--into`); every subcommand returns "not implemented" on stderr with exit code 1 (except `--help`/`--version`).
- Bare `undo` (no subcommand) and `undo <OP_ID>` positional must parse (use `args_conflicts_with_subcommands` + optional positional).

## Detailed Requirements

1. Dependencies limited to the ADR-003 allowlist; only `clap` needed now.
2. `undo --version` prints `undo <semver>` from `CARGO_PKG_VERSION`; exit 0.
3. Exit code for unknown flags/subcommands must be 2 (clap default configured accordingly) per DESIGN §3.3.
4. `undo __hook` must be hidden from `--help` output.
5. `cargo build --release` produces a single self-contained binary; no build.rs.
6. Add `justfile` or `Makefile` with `build`, `test`, `fmt`, `lint` targets (pick `justfile`; document commands in a top-level comment).

## Acceptance Criteria

- [ ] `cargo build` and `cargo build --release` succeed on macOS arm64.
- [ ] `undo --help` lists exactly the DESIGN §3.2 user-facing subcommands; `__hook` absent from help but invocable.
- [ ] `undo --version` exit 0; `undo nosuchcmd` exit 2; `undo list` exit 1 with `not implemented`.
- [ ] `undo 01JZK3QG4Y9Z8XWVUTSRQPNMKJ` parses as op-id positional (stub), not as a subcommand error.
- [ ] Module files exist with DESIGN-section doc comments.

## Validation

```sh
cargo fmt --check && cargo clippy --all-targets -- -D warnings && cargo test
./target/release/undo --version
./target/release/undo __hook --help >/dev/null 2>&1; test $? -ne 2 || true  # invocable
```

Add `tests/cli_skeleton.rs` using `assert_cmd` asserting the exit codes above.

## Dependencies

None.

## Non-goals

Any real behavior; CI (issue 02); error taxonomy (issue 03).

## Design References

DESIGN.md §3.2, §3.3, §4; ADR-003.
