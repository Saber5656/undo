# ADR-006: Store Security Model — Same-User Trust Boundary, No Encryption, No Network

- Status: Accepted
- Date: 2026-07-07
- Related: DESIGN.md §12, §10

## Context

The store necessarily contains **copies of whatever the user deletes or overwrites**, including credentials and private documents. It extends the lifetime of sensitive data beyond the user's deletion intent. We must define the trust boundary, at-rest posture, and data-lifetime controls for a public OSS release.

## Decision

1. **Trust boundary = the macOS user account.** The store is `0700` (dirs) / `0600` (files), owned by the invoking user. On every store open, the core verifies owner and permissions and **refuses with exit code 5** (never auto-chmods) if they are wrong.
2. **No encryption at rest in v1.** The threat actor who can read the store can read the original files too; a local key would live in the same account and add no real barrier. macOS FileVault covers the disk-theft case. Documented plainly in SECURITY.md.
3. **Bounded data lifetime by default**: retention defaults `max_age_days = 7`, `max_ops = 500`, `max_total_bytes = 2GiB` (logical), enforced by GC (auto + manual). `undo purge <id>|--all` gives immediate manual disposal. Purge documentation states honestly that deletion on APFS/SSD is *not* forensic shredding.
4. **The store never leaves the machine**: Time Machine exclusion is set on the store root at init (per-path exclusion, no root needed); Spotlight is suppressed via `ops.noindex`. This prevents deleted secrets from silently entering backups and search indexes.
5. **Zero network capability, zero telemetry, no auto-update.** Enforced structurally: no network crate may enter the dependency graph (`cargo-deny` ban list, CI-gated; ADR-003).
6. **Command text privacy**: intercepted command lines (which may embed secrets, e.g. `mysql -pPASS`) are passed to the core via **stdin, never argv** (argv is world-readable via `ps`). Manifests storing command text inherit store permissions.
7. **Parser is a no-eval boundary**: the core never executes or shell-evaluates any part of the observed command text (ADR-001); this is fuzz-tested.

## Alternatives considered

- **Encrypted store (age/keychain-wrapped key)**: rejected for v1 — no meaningful attacker it stops within the same-user boundary, real UX cost (restore needs unlock), key management complexity. Revisit in v2 only with a concrete threat model (e.g. shared-admin machines).
- **Per-volume stores (`.undo-store` at volume roots, Finder-Trash style)**: rejected for v1 — write-permission variance and polluting external media; cross-volume operands use the capped copy fallback instead.
- **Default-exclude sensitive paths (`~/.ssh`, keychains)**: rejected — deleting an SSH key is *exactly* the accident users want undone. Exclusions remain a user-configurable list with minimal defaults (store itself, `~/Library/Caches`, `~/.Trash`).

## Consequences

- SECURITY.md (issue 30) documents: threat model, what the store contains, retention knobs, purge semantics, FileVault recommendation, and the no-network guarantee.
- `undo doctor` checks store ownership/permissions, TM-exclusion xattr presence, and warns on world-readable ancestors.
- Restore paths are validated against symlink swaps (`O_NOFOLLOW`, parent canonicalization — DESIGN.md §12.4); op-id CLI inputs are validated against the ULID alphabet to prevent store path traversal.
