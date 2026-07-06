# Title

cargo-audit/deny gates, SECURITY.md

## Summary

Add supply-chain CI gates (cargo-audit advisories, cargo-deny licenses/bans including the network-crate ban) and write SECURITY.md documenting the threat model, data-at-rest posture, and reporting process.

## Context

ADR-006 promises a structurally-enforced no-network guarantee; ADR-003 fixes the dependency allowlist; DESIGN §12.6–12.8 define what SECURITY.md must cover for a public OSS release.

## Scope

- `deny.toml`: 
  - `[bans]` deny-list at minimum: `reqwest`, `hyper`, `ureq`, `curl`, `isahc`, `attohttpc`, `tokio` (full), `async-std`, `libssh2-sys`, `native-tls`, `rustls` — with a comment: "undo must have no network capability (ADR-006)"; wildcard-version entries;
  - `[licenses]` allowlist: MIT, Apache-2.0, BSD-2/3-Clause, ISC, Unicode-3.0 (adjust to actual graph with justification comments);
  - `[advisories]` deny on vulnerabilities; `[sources]` crates.io only.
- CI jobs (extend 02): `cargo deny check` and `cargo audit` (advisory DB cached; audit failures on *warnings* allowed to pass with `--deny unmaintained` NOT set — only vulnerability class blocks) on every PR + weekly schedule.
- `SECURITY.md` sections (concrete content, not placeholders): supported versions; reporting channel (GitHub private vulnerability reporting — enable in repo settings: include the exact settings path as a maintainer note); threat model summary (user-account trust boundary, what the store contains, ADR-006 items 1–7 in prose); no-network guarantee and how CI enforces it; secrets-lifetime guidance (`retention`, `purge`, FileVault recommendation); parser boundary + fuzzing note; release integrity (checksums, provenance — forward-reference issue 33).
- Root `docs/` link check: SECURITY.md linked from README (touch README's security section stub only; full README is issue 32 — coordinate ordering, or land the link in 32 if it merges later; put the canonical text here).

## Detailed Requirements

1. `cargo deny check` must pass on the current dependency graph — if any allowlisted crate transitively pulls a banned/unknown-license crate, resolve (feature-trim or replace) within this issue and update ADR-003's table if the allowlist changes.
2. A deliberate test: add `ureq` as a dev-dependency in a scratch branch → CI fails (screenshot/log in PR, then revert).
3. SECURITY.md statements must match implemented behavior exactly (cross-check against ADR-006 and issues 05/28 outputs; the purge honesty line reuses 28's exact wording).

## Acceptance Criteria

- [ ] CI shows green `deny`/`audit` jobs; the ban-verification experiment documented.
- [ ] SECURITY.md complete per the section list; no TODO/placeholder text.
- [ ] Weekly scheduled run configured.

## Validation

CI links in PR; markdown-lint (default GitHub rendering) clean.

## Dependencies

02.

## Non-goals

README rewrite (32), release attestation implementation (33), enabling the GitHub setting itself (maintainer/manual — noted as handoff).

## Design References

DESIGN.md §12.6–§12.8; ADR-003; ADR-006.
