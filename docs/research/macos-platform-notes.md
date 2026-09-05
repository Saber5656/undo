# Research: macOS / zsh Platform Facts Load-Bearing for the Design

- Date: 2026-07-07
- Status: informs ADR-001, ADR-002, DESIGN.md §5–§7, §14
- Method: official documentation (zshmisc, clonefile(2) man page), Apple engineering commentary, GitHub Actions runner documentation

## 1. zsh `preexec` hook semantics (zshmisc)

> Executed just after a command has been read and is about to be executed. If the history mechanism is active [...], the string that the user typed is passed as the first argument, otherwise it is an empty string. The actual command that will be executed (including expanded aliases) is passed in two different forms: the second argument is a single-line, size-limited version of the command (with things like function bodies elided); the third argument contains the full text that is being executed.

Consequences for the design:

- `$3` gives us the full text **with aliases expanded** — the hook forwards `$3` (falling back to `$1` when empty) to the core binary.
- `$3` is **not** glob-expanded and **not** parameter-expanded. The core must do its own conservative glob/tilde expansion and must treat `$var`, `$(...)`, backticks, and `${...}` as *unresolvable* (never evaluate them — security boundary, DESIGN.md §12).
- `preexec` runs for interactive shells only; non-interactive scripts are out of scope for v1 by construction.
- Whether `preexec` fires before zsh's own `no matches found` glob abort is version-dependent and harmless either way (worst case: a snapshot for a command that never ran). Listed as a known unknown.

## 2. `clonefile(2)` — file clones are the fast path; directory clones are discouraged

From the man page and Apple engineering commentary (2026):

- `clonefile()` creates a copy-on-write clone: `dst` shares data blocks with `src`; attributes, xattrs and ACLs are copied. Writes to either side are private. Introduced in macOS 10.12; APFS supports it on essentially all modern macOS boot volumes.
- `CLONE_NOFOLLOW` prevents following `src` when it is a symlink (files only).
- Volume support is testable via `getattrlist(2)` → `ATTR_VOL_CAPABILITIES` → `VOL_CAP_INT_CLONE`. (`undo doctor` uses this.)
- **Directory cloning works but Apple strongly discourages it** ("Use copyfile(3) instead"): the kernel performs the clone atomically while locking the source hierarchy, and Apple engineers state that a sufficiently large/contended hierarchy can stall the kernel long enough to panic. It also does not provide true directory-level dedup semantics — items are cloned as if individually.

Consequences (ADR-002):

- Snapshot backend clones **individual files** with `clonefile(2)` and walks directories itself (bounded by entry caps and a soft deadline). It never calls `clonefile()` on a directory.
- Cross-volume or non-cloneable targets (`EXDEV`, `ENOTSUP`) fall back to a bounded byte-copy.
- Clones cost ~0 bytes until the original's blocks are freed (deletion) or rewritten (`sed -i`), so same-volume snapshots are effectively free at snapshot time; retention accounting uses logical bytes (honest upper bound).

## 3. `copyfile(3)`

`copyfile(3)` with `COPYFILE_CLONE` does clone-else-copy in one call, but does not report which path was taken, which defeats our copy-size caps. We therefore call `clonefile(2)` directly and implement the copy fallback ourselves (ADR-002).

## 4. GitHub Actions macOS runners (as of 2026-07)

- Available labels: `macos-15` (arm64), `macos-26` (arm64), `macos-26-intel` (x64). `macos-latest` re-points to `macos-26` between 2026-06-15 and 2026-07-15.
- `macos-14` began deprecation on 2026-07-06 — do not target it.
- Runners use APFS volumes, so `clonefile` integration tests run natively in CI. Cross-volume (`EXDEV`) tests can use a RAM disk (`hdiutil attach -nomount ram://…` + `newfs_hfs`) to get a second, non-cloneable volume.
- CI matrix decision: `macos-15` + `macos-26` (arm64); Intel binaries are produced by cross-compiling (`x86_64-apple-darwin` target) and joined with `lipo` for the universal release artifact.

## 5. BSD userland quirks that affect analyzers

- macOS `sed` requires `-i <ext>` as a **separate argument** (`sed -i '' file` for "no backup"); GNU `gsed` (Homebrew) takes an optional **attached** suffix (`gsed -i file`). The in-place analyzer must implement both grammars keyed by command name (DESIGN.md §6.7).
- Homebrew installs GNU coreutils with a `g` prefix (`grm`, `gmv`, `gcp`, `gsed`, `gchmod`, `gchown`); the command table maps these to the same analyzers.
- `mv`/`cp` on macOS are BSD variants; flag sets used by the analyzers are enumerated in DESIGN.md §6 against the macOS man pages.

## 6. Miscellaneous platform facts

- macOS ships zsh 5.9 as `/bin/zsh` on all supported versions (Ventura+). Minimum supported: zsh ≥ 5.8, macOS ≥ 13.
- TCC: a terminal without Full Disk Access may be denied on `~/Desktop`, `~/Documents`, etc. The snapshot engine treats permission errors as warn-and-proceed (ADR-005); `undo doctor` surfaces an FDA hint.
- Spotlight indexing of a directory is suppressed by a `.noindex` name suffix; Time Machine exclusion can be set programmatically per-path (no root required). Both are applied to the store (ADR-006).
- APFS default volumes are case-insensitive: the tool never case-folds paths itself and compares byte-exact paths only.

## Sources

- https://manpages.debian.org/testing/zsh-common/zshmisc.1.en.html (preexec)
- https://www.manpagez.com/man/2/clonefile/
- https://mjtsai.com/blog/2026/05/14/apfs-folder-clones/ (Apple commentary on directory clones)
- https://eclecticlight.co/2020/04/14/copy-move-and-clone-files-in-apfs-a-primer/
- https://github.blog/changelog/2026-02-26-macos-26-is-now-generally-available-for-github-hosted-runners/
- https://github.com/actions/runner-images (macOS images; macos-14 deprecation notice)
