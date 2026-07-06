# ADR-002: Per-File APFS Clones with Bounded Walk + Copy Fallback

- Status: Accepted
- Date: 2026-07-07
- Related: DESIGN.md §7, [macos-platform-notes](../research/macos-platform-notes.md)

## Context

Snapshots must be fast enough to run synchronously inside `preexec` (budget: low tens of milliseconds for typical operations) and must not risk system stability. Candidates:

1. **`clonefile(2)` on the whole directory** — one syscall, kernel-atomic
2. **Own directory walk + `clonefile(2)` per file** (+ `mkdir`/`symlink` recreation)
3. **`copyfile(3)` with `COPYFILE_CLONE`** (clone-else-copy in one call)
4. **Plain byte copies** into the store
5. **Move-to-trash instead of letting the command delete** (trash-cli model)
6. **APFS volume snapshots** (`tmutil localsnapshot`)
7. **git-style content-addressed object store**

## Decision

Use **(2)**: the snapshot engine walks directories itself (bounded by `max_entries_per_op` and a soft deadline), clones regular files with `clonefile(2)` (`CLONE_NOFOLLOW`), recreates directories and symlinks structurally, and falls back to a **bounded byte copy** (4) per file when cloning is impossible (`EXDEV`, `ENOTSUP`, non-APFS volumes).

## Rationale

- **(1) is explicitly discouraged by Apple** ("Use copyfile(3) instead"): the kernel locks the source hierarchy for the duration and Apple engineers state large/contended hierarchies can stall long enough to **panic the kernel**. A safety tool must not carry a kernel-panic tail risk.
- **(3)** hides whether a clone or a copy happened, defeating our cross-volume copy-size caps; calling `clonefile(2)` directly keeps the fallback under our control and caps.
- **(2) keeps clone economics**: on APFS a file clone shares all data blocks — snapshot cost is metadata-only regardless of file size, so even multi-GB files snapshot in microseconds. The walk cost is proportional to entry count, which the caps/deadline bound explicitly (DESIGN.md §7.5) with a warn-not-protected outcome instead of unbounded latency.
- **(5) alters command semantics** (the file is no longer deleted by `rm` but moved), violating ADR-001's observer principle, and does not generalize to overwrites (`sed -i`, `cp`) or metadata changes.
- **(6)** requires root for mount-based restore workflows, is whole-volume (privacy/scope), and has no per-command granularity. Rejected for v1; noted as a v2 idea for atomicity of giant trees.
- **(7)** adds hashing I/O on the hot path (reads whole file contents — exactly what clones avoid) and store complexity with no v1 benefit.

## Consequences

- Same-volume snapshots are near-free in bytes at snapshot time; they become "real" bytes only when originals are deleted or rewritten. Retention accounting therefore uses **logical bytes** as the honest upper bound (DESIGN.md §10).
- Payload clones preserve xattrs and ACLs automatically (clone semantics), so content restores are high-fidelity without extra code.
- Sockets, FIFOs, and device nodes inside snapshotted trees are recorded but not payload-captured (DESIGN.md §13 edge table).
- The store must live on the user's home volume for the common-path clone to work; cross-volume operands transparently use the capped copy fallback.
