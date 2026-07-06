# ADR-003: Rust Implementation with a Minimal Dependency Allowlist

- Status: Accepted (language choice confirmed by the product owner, 2026-07-07)
- Date: 2026-07-07
- Related: DESIGN.md §4, §12.8

## Context

The core binary sits on the interactive-shell hot path (spawned from `preexec`), performs low-level filesystem work (`clonefile(2)`, `lstat`, `O_NOFOLLOW` opens), and will be published as open source. Implementation language candidates were Rust, Go, and shell-plus-helper.

## Decision

Implement the entire core as a **single Rust binary** (`undo`). Dependencies are restricted to an explicit allowlist; additions require an ADR update or a documented justification in the PR.

Allowlist (v1):

| Crate | Purpose |
|---|---|
| `clap` (v4, derive) | CLI parsing |
| `serde`, `serde_json` | Manifest/index/JSON output |
| `toml` | Config parsing |
| `ulid` | Operation ids (time-sortable) |
| `libc` | `clonefile`, `lstat`, `flock`, `openat`, `getattrlist` |
| `glob` | Conservative glob expansion (with zsh-default match options) |
| `thiserror` | Error taxonomy |
| dev-only: `assert_cmd`, `predicates`, `tempfile`, `cargo-fuzz` (tooling) | Testing/fuzzing |

Explicitly banned at CI level (`cargo-deny`): any network stack (`reqwest`, `hyper`, `curl`, …) — the tool must have **no network capability** (ADR-006).

## Rationale

- Startup latency of a Rust binary (~a few ms) fits the `preexec` budget; Go is acceptable too but its runtime start and binary size are slightly worse, and raw syscall work (`clonefile`) is more natural via `libc` FFI in Rust.
- Memory-safety matters: the parser consumes untrusted command text (security boundary, DESIGN.md §12.1) and is fuzzed.
- Shell-based implementation cannot meet the testing, fuzzing, and robustness bars this design requires.
- A small allowlist keeps the supply-chain review surface auditable for an OSS security-adjacent tool.

## Consequences

- `rust-toolchain.toml` pins `channel = "stable"`; `Cargo.lock` is committed; `cargo audit` + `cargo deny` run in CI (issue 30).
- Universal macOS binaries are produced by building `aarch64-apple-darwin` + `x86_64-apple-darwin` and joining with `lipo` (issue 33).
- No async runtime: all I/O is synchronous by design (hot path is a single small task; simplicity wins).
