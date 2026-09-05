# Title

Store layout, init, permission checks, locking

## Summary

Implement the on-disk store per ADR-004: directory layout, first-run initialization with secure permissions and Time Machine exclusion, owner/permission verification with refusal semantics, format versioning, and the flock-based locking primitives.

## Context

DESIGN §8.1/§8.4 define layout and lock roles; ADR-006 defines the refusal posture (never auto-fix). Snapshot (10), restore (25), GC (27), purge (28) all consume these primitives.

## Scope

- `src/store/layout.rs` (+ `store/lock.rs`): `Store::open_or_init(root: PathBuf) -> Result<Store>`.
- Root resolution: `--store` > `UNDO_STORE` > `${XDG_DATA_HOME:-~/.local/share}/undo`.
- Init (first run): create root + `ops.noindex/` with mode 0700; `format-version` containing `1\n` (0600); `lock` (0600); set Time Machine exclusion on root via the per-path exclusion xattr (`com.apple.metadata:com_apple_backup_excludeItem` with the documented plist value); failures to set the xattr are a warning, not an error.
- Open (subsequent): verify root owner == euid and mode has no group/other bits; violation → `StoreIntegrity` error (exit 5) with remediation text showing exact `chmod 700` / `chown` commands; never modify permissions itself.
- `format-version` parse: unknown major → exit-5 error `store requires a newer undo (format N)`.
- Locking API: `Store::lock_exclusive() -> Guard`, `lock_shared() -> Guard`, `try_lock_exclusive()` — flock(2) on `<store>/lock`, with a 2 s bounded retry loop (50 ms steps) for the blocking variants; RAII unlock.
- Helper: `Store::ops_dir()`, `Store::op_path(id)`, `Store::st_dev()` (cached lstat of root).

## Detailed Requirements

1. All directories/files created with explicit modes via `libc` (umask-independent): dirs 0700, files 0600.
2. Owner check uses `lstat` uid vs `geteuid`.
3. TM-exclusion xattr value must match what `tmutil isexcluded` recognizes (verify in test via `tmutil isexcluded <root>` output containing `[Excluded]`).
4. Concurrent `open_or_init` from two processes must both succeed (mkdir EEXIST tolerated) — test with two spawned processes.
5. Lock guards must release on drop and on process death (flock semantics); shared+exclusive interplay tested (EX blocks SH and vice versa; two SH coexist).

## Acceptance Criteria

- [ ] Fresh init produces exactly the DESIGN §8.1 tree (minus ops) with correct modes (asserted via lstat in tests).
- [ ] Store with mode 0755 or foreign-looking owner (simulate by chmod in test) → open fails exit-5 style error containing `chmod 700`.
- [ ] `tmutil isexcluded` reports the store excluded (integration test; skip gracefully if `tmutil` absent).
- [ ] Newer `format-version` (e.g. `2`) → refusal with the exact message above.
- [ ] Lock semantics tests pass (two-process EX/SH matrix).

## Validation

`cargo test store::layout store::lock` green on macOS CI (real APFS tempdirs under `$TMPDIR`).

## Dependencies

03.

## Non-goals

Manifests/index (06); GC sweeping (27); doctor reporting (21).

## Design References

DESIGN.md §8.1, §8.4, §12.6; ADR-004; ADR-006.
