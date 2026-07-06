# Title

Homebrew tap formula and publication handoff

## Summary

Create the Homebrew formula for `undo` (binary formula pointing at the GitHub Release universal tarball), the tap-repository content, an update script for future releases, and the documented handoff for the steps only the repository owner can perform.

## Context

Distribution decision includes a Homebrew tap. The tap lives in a separate repository (`<owner>/homebrew-tap`); creating that repo and pushing to it are **product-owner manual steps** (agents must not create repositories or set credentials — handoff per project rules). U6 (formula shape for universal binaries) is resolved here.

## Scope

- In THIS repo: `packaging/homebrew/undo.rb` — formula template:
  - `url` → versioned release tarball; `sha256` from SHA256SUMS; universal binary → single `url` (no per-arch blocks needed — resolve U6 by testing; if brew audit complains, switch to `on_arm`/`on_intel` with the same universal tarball and document);
  - `def install: bin.install "undo"`;
  - `test do`: `assert_match version.to_s, shell_output("#{bin}/undo --version")` + `system bin/"undo", "init", "zsh"` exits 0;
  - `caveats`: the `eval "$(undo init zsh)"` line + `undo doctor` pointer.
- `packaging/homebrew/README.md` — handoff runbook for the owner:
  1. create public repo `<owner>/homebrew-tap` (exact steps, `gh repo create` command included but to be run by the owner);
  2. copy `Formula/undo.rb`; commit;
  3. verification: `brew tap <owner>/tap && brew install undo && brew test undo && undo doctor`;
  4. per-release update procedure (url/sha bump) + the provided `scripts/update-tap-formula.sh` (reads a release tag, downloads SHA256SUMS, prints/applies the formula diff; runs `brew style`/`brew audit --strict` locally).
- `scripts/update-tap-formula.sh` implemented + shellcheck-clean.
- CI (optional job, PR-path-filtered): `brew style packaging/homebrew/undo.rb` + `brew audit --formula` in stub mode against the dry-run artifact when accessible; otherwise `brew style` only (audit needs a live URL — gate on release existence).

## Detailed Requirements

1. Formula must pass `brew style` and `brew audit --strict --online` against the first real release (record transcript in the handoff completion note).
2. No credentials, tokens, or repo-creation performed by the implementing agent — the runbook stops at command listings for the owner (per repository security rules).
3. `update-tap-formula.sh` verifies the tarball sha against the attestation (`gh attestation verify`) before emitting the formula bump (supply-chain link to 33).
4. End-to-end validation plan (post-owner-handoff): fresh macOS machine/VM or CI `brew install <owner>/tap/undo` → `undo doctor` exit 0 → an `rm`/restore round-trip.

## Acceptance Criteria

- [ ] `brew style` clean; formula installs from a local tarball via `brew install --build-from-source ./packaging/homebrew/undo.rb`-style local test or documented equivalent.
- [ ] Runbook executable by the owner without further questions (dry-walkthrough recorded).
- [ ] Update script tested against the 33 dry-run artifact (sha + attestation verification demonstrated).

## Validation

Transcripts in PR; owner handoff note filed when the tap goes live.

## Dependencies

33 (release artifact + SHA256SUMS + attestation).

## Non-goals

homebrew-core submission (v2+ after adoption), Linuxbrew, cask.

## Design References

DESIGN.md §3.1, §12.8, §17 U6; product decision (distribution) in DESIGN §1.1.
