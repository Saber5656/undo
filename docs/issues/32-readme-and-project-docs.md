# Title

README, CONTRIBUTING, LICENSE

## Summary

Write the public-facing README (English), CONTRIBUTING.md, and add the license files (MIT OR Apache-2.0 dual), replacing the placeholder README.

## Context

The repo is public from day one; README is the product's front page and must state the coverage contract honestly (ADR-005: documented false-negative surface builds trust). License: Rust-ecosystem dual MIT/Apache-2.0 is the design default — **requires product-owner confirmation before merge** (explicit gate below).

## Scope

- `README.md` (English) sections: hero one-liner + demo block (the §3.5 session, updated to real output); why/what (observer model, one paragraph + the prior-art table condensed); install (brew tap + GitHub Releases + `undo init zsh` line + `undo doctor`); what's protected / what's NOT protected (the §6.4 not-protected list verbatim as a user-facing table — normative honesty section); how it works (3-sentence architecture + link to docs/DESIGN.md); configuration (key table condensed from §11); data & privacy (store location, retention defaults, purge, no-network — link SECURITY.md); uninstall (remove eval line, `undo purge --all`, delete store/config); limitations & roadmap (v2 items from §16); badges (CI, license); Japanese README deferred (v2, §16).
- `CONTRIBUTING.md`: dev setup (`rust-toolchain`, `just` targets), test layers overview (§15 table condensed), how to run E2E locally (RAM-disk helper note), dependency policy (ADR-003 allowlist + amendment process), security-sensitive areas requiring extra review (analyze/, restore write paths), conventional commit style, docs-first rule (DESIGN.md is canonical; behavior changes update DESIGN + issues first).
- `LICENSE-MIT`, `LICENSE-APACHE`, README license section, `Cargo.toml` `license = "MIT OR Apache-2.0"`.
- Keep the original Japanese one-liner as the repo description tagline inside README's hero (bilingual hero line acceptable).

## Detailed Requirements

1. Every claim in README must be true of the merged code at the time this issue lands (verify against `--help`, doctor output, config defaults — no aspirational features).
2. The not-protected table must enumerate at minimum: scripts/non-interactive, `$VAR`/`$(…)` args, `xargs`/`find -delete`, redirection truncation, `sudo`, git ops — mirroring §6.4/§2.2 wording.
3. **Gate**: obtain explicit product-owner (repository owner) approval of the dual-license choice in the issue/PR thread before merging; if a different license is chosen, update `Cargo.toml`, files, and ADR-003 accordingly.
4. All internal doc links relative and valid (CI-agnostic; verify with a link-check script or manual pass documented in PR).

## Acceptance Criteria

- [ ] README renders correctly on GitHub (screenshot in PR) with working demo block and links.
- [ ] Not-protected section present and §6.4-complete.
- [ ] License approval recorded (comment link) + files/Cargo metadata consistent.
- [ ] CONTRIBUTING covers dev-setup-to-green-tests on a fresh machine (validated by following it verbatim once — note in PR).

## Validation

Markdown renders clean; link check pass; fresh-clone CONTRIBUTING walkthrough executed.

## Dependencies

26 (accurate UX), 30 (SECURITY.md to link), 21 (doctor wording).

## Non-goals

Docs website, README.ja.md, logo/branding, changelog automation (33 covers release notes).

## Design References

DESIGN.md §2.2, §3.5, §6.4, §11, §16; ADR-003; ADR-005; research/prior-art.md.
