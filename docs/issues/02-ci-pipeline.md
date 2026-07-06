# Title

CI pipeline: fmt, clippy, tests on macOS runners

## Summary

Add a GitHub Actions workflow running formatting, lint, and test gates on `macos-15` and `macos-26`, with dependency caching.

## Context

All later issues rely on these gates. Runner labels per research/macos-platform-notes.md §4 (`macos-14` is deprecating; do not use). Rust-only repo; no cross-platform matrix.

## Scope

- `.github/workflows/ci.yml` triggered on `pull_request` and `push` to `main`.
- Jobs: `lint` (fmt --check, clippy `--all-targets -- -D warnings`) on `macos-15`; `test` matrix on `[macos-15, macos-26]` running `cargo test --all-targets`.
- Cache: `~/.cargo/registry`, `~/.cargo/git`, `target/` keyed on `Cargo.lock` + toolchain (use `Swatinem/rust-cache@v2`).
- Minimal workflow permissions: `permissions: { contents: read }` at workflow level.
- Concurrency group cancelling superseded runs per ref.

## Detailed Requirements

1. Toolchain from `rust-toolchain.toml` (no explicit version in workflow).
2. Pin third-party actions by major tag at minimum; prefer SHA pins for non-GitHub-owned actions (rust-cache) with a comment naming the version.
3. Workflow must fail on any warning (`-D warnings`) and on `cargo fmt --check` diff.
4. Add a status badge to the top of `README.md` (do not otherwise rewrite README — issue 32 owns it).
5. Job timeout ≤ 20 minutes.

## Acceptance Criteria

- [ ] CI runs on a PR touching any file and reports the three gates as separate checks (lint, test macos-15, test macos-26).
- [ ] A deliberately mis-formatted commit fails `lint` (verify once in the PR, then fix).
- [ ] Second CI run on the same lockfile restores caches (observe cache-hit in logs).
- [ ] `GITHUB_TOKEN` permissions are read-only for contents.

## Validation

Push a branch; verify all checks green in the PR UI. Include the run URL in the implementation PR description.

## Dependencies

01.

## Non-goals

Fuzz jobs (29), audit/deny (30), E2E jobs (31), release workflow (33).

## Design References

DESIGN.md §15; research/macos-platform-notes.md §4.
