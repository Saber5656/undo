# Title

clonefile(2) snapshot backend

## Summary

Implement the same-volume snapshot backend: safe `clonefile(2)` FFI for regular files with `CLONE_NOFOLLOW`, plus the bounded, symlink-safe directory walk that recreates trees in the payload area (per-file clones; never directory clones).

## Context

ADR-002 mandates per-file clones with our own walk because Apple strongly discourages directory clonefile (kernel-panic tail risk). This backend is the hot path — same-volume snapshots must be metadata-cost only.

## Scope

- `src/snapshot/clone.rs`: `fn clone_file(src: &Path, dst: &Path) -> Result<(), CloneError>` via `libc::clonefile` (fall back to declaring the extern if the libc version lacks it) with `CLONE_NOFOLLOW`; error taxonomy distinguishing `EXDEV`/`ENOTSUP` (→ caller falls back to copy), `EACCES`/`EPERM` (walk-error), `ENOSPC` (abort-op class), other.
- `src/snapshot/walk.rs`: `fn capture_tree(src_dir, payload_dst, budget) -> TreeReport` — depth-first walk using `openat`-anchored traversal (`O_NOFOLLOW | O_DIRECTORY` per component; DESIGN §12.4):
  - directories → `mkdirat` in payload (record mode/uid/gid/mtime per node into the report);
  - regular files → `clone_file` (caller-provided fallback closure for EXDEV/ENOTSUP, wired in issue 10);
  - symlinks → `symlinkat` recreating the identical target string, never resolved;
  - sockets/FIFOs/devices → recorded `special`, skipped (§7.3);
  - hard links → captured as independent files (F15).
- `budget`: entry counter (shared with §7.5 caps) + soft-deadline clock checked every 64 entries; on exhaustion return partial report with `stopped_reason`.
- Volume clone-capability probe: `fn volume_supports_clone(path) -> bool` via `getattrlist` `VOL_CAP_INT_CLONE` (used by doctor, 21).

## Detailed Requirements

1. All metadata reads `lstat`/`fstatat(AT_SYMLINK_NOFOLLOW)`.
2. A component that becomes a symlink mid-walk (swap attack) must produce a per-node `walk-error`, not an escape (assert via a test that pre-plants a symlink where a dir is expected).
3. Clone of a cloned file's tree must preserve xattrs (assert one xattr survives capture).
4. `TreeReport` carries: nodes captured, per-node attr records (for manifest entries), logical bytes (sum of file `st_size`), special/skipped nodes with reasons.
5. No recursion deeper than 512 components (explicit depth guard → walk-error).
6. iCloud-dataless probe (U2): a unit test is not feasible in CI; add a `#[ignore]`d manual test + document observed behavior in the PR description.

## Acceptance Criteria

- [ ] Cloning a 1 GiB sparse test file completes < 50 ms on CI (loose assert ≤ 500 ms) and occupies ~0 additional blocks (assert via `st_blocks` of clone < source's full block count is not reliable — instead assert wall-time only and content equality by hash of first/last MiB).
- [ ] Tree capture reproduces: nested dirs, files with xattrs, symlink (dangling and valid), FIFO recorded-skipped; modes/uid/gid recorded per node.
- [ ] Symlink-swap test yields walk-error and no payload outside the payload dir.
- [ ] EXDEV path (RAM-disk source per research §4) surfaces `CloneError::CrossDevice` for the fallback.
- [ ] Entry-cap and deadline exhaustion return partial `TreeReport` with `stopped_reason`.

## Validation

`cargo test snapshot::clone snapshot::walk` on CI (APFS `$TMPDIR`); RAM-disk fixture helper script `scripts/mk-ramdisk.sh` committed (hdiutil, per research §4) and used by tests guarded with `#[cfg(target_os = "macos")]` + runtime skip if hdiutil fails.

## Dependencies

05.

## Non-goals

Copy fallback internals (08), backend selection & manifest assembly (10), metadata-only entries (09).

## Design References

DESIGN.md §7.2, §7.3, §7.5, §12.4, §13 (F13–F15); ADR-002; research/macos-platform-notes.md §2–3.
