# ADR-004: Filesystem Store with Per-Operation JSON Manifests (No Database)

- Status: Accepted
- Date: 2026-07-07
- Related: DESIGN.md §8, §10

## Context

Each intercepted command produces one *operation*: metadata plus zero or more payload snapshots. The store needs: crash-safe concurrent writes from multiple shells, time-ordered listing, retention accounting, and integrity checking. Candidates: SQLite (`rusqlite`), a single JSONL log, or per-operation directories with JSON manifests.

## Decision

Use **per-operation directories** named by ULID under `<store>/ops.noindex/`, each containing `manifest.json` (authoritative record) and `payload/` (snapshots). A single append-only `<store>/index.jsonl` acts as a **rebuildable cache** for fast listing and size accounting. Commit protocol: write into `<id>.staging/`, fsync manifest, rename to `<id>/` (atomic commit point).

Store root: `${XDG_DATA_HOME:-~/.local/share}/undo/` (overridable via `UNDO_STORE`).

## Rationale

- **Concurrency without coordination**: two shells snapshotting simultaneously write disjoint ULID directories — no shared write lock on the hot path. SQLite would serialize writers and add locking/corruption-recovery complexity; a single JSONL log would need locking for every append *and* cannot hold payloads.
- **Crash safety by construction**: a crash leaves only a `.staging` directory, which GC sweeps after a grace period; committed operations are always internally complete.
- **ULID directory names sort lexicographically = chronologically**, so "most recent operation" and age-based GC need no index at all; `index.jsonl` is purely an optimization and can always be rebuilt by scanning (self-healing on corruption).
- **Human-inspectable**: users can audit exactly what the tool stored with `ls` and `cat` — important for trust in a tool that copies their files.
- No query patterns in the CLI need more than "list recent / get by id", so a database buys nothing.

## Consequences

- `manifest.json` gets an explicit `"format": 1` version; the binary refuses stores with a newer major format (clear error, exit code 5).
- `index.jsonl` appends take a short exclusive `flock`; readers tolerate a stale index and fall back to scanning.
- Payloads are named by entry index (`payload/<n>`), never by original filename — immune to hostile filenames (control chars, newlines, path separators) (DESIGN.md §12.5).
- Spotlight indexing is suppressed via the `ops.noindex` directory name; Time Machine exclusion is set on the store root at init (ADR-006).
