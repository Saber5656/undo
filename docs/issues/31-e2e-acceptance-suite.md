# Title

End-to-end acceptance scenario suite

## Summary

Build the product-level acceptance harness: scripted real-zsh sessions against a release-profile binary executing the full scenario matrix (interception → destruction → restore round-trips, failure modes, lifecycle), run in CI as the v1 exit gate.

## Context

DESIGN §15 L3 and §6 (v1 completion gate in ISSUE_PLAN). Unit/integration layers prove modules; this suite proves the *product*: hook + binary + store on a real shell, including the §13 behaviors that only manifest end-to-end.

## Scope

- `tests/e2e/` harness (Rust test binary or `tests/e2e.rs` + helper module) using the L2 zsh harness from issue 20 (direct `zsh -ic` or zpty fallback), a fresh `HOME`/`ZDOTDIR`/`UNDO_STORE` sandbox per scenario, and the **release-profile** binary (`cargo build --release` in CI job).
- Scenario matrix (each = setup → shell command(s) → assertions on fs + store + restore round-trip). Minimum set:
  1. `rm file` → restore → byte-identical (hash) + mode/mtime;
  2. `rm -rf tree` (nested, symlink inside, xattr file) → restore → tree-hash identical, symlink target string preserved, xattr present;
  3. `rm *.log` glob (+dotfile invariant: `.hidden.log` untouched by snapshot);
  4. `mv a b` (b existed) → restore → both a and b back;
  5. `mv a dir/` → restore;
  6. `cp big b` (b existed) → restore b's pre-image;
  7. `cp -R src dst` partial collisions → restore collided files only;
  8. `sed -i '' 's/x/y/' f` → restore pre-image;
  9. `perl -pi -e …` → restore;
  10. `chmod -R 700 tree` → restore modes (spot-check 3 nodes);
  11. `chown :staff f` (same-user group change) → restore;
  12. line merge: `rm a; chmod 600 b` → ONE op; restore both effects;
  13. alias `rm='rm -v'` defined in fixture zshrc (F18) → intercepted;
  14. `sudo rm` → notice, no op (use `sudo -n` guard: skip scenario when sudo needs a password — CI has passwordless sudo);
  15. `rm $F` unresolvable → notice, no op, command ran;
  16. conflict path: `rm f` → recreate f manually → `undo` exit 3; `undo --force` replaces + inverse op exists;
  17. restore-of-restore returns to post-command state;
  18. `--into` extraction;
  19. cap: 60k-entry tree with default caps → partial/aborted op + warn + command proceeded (tree built by script; generous CI timeout);
  20. copy-fallback: RAM-disk file rm → snapshot via copy → restore;
  21. ENOSPC-ish: store on tiny RAM disk → warn `store unavailable`/abort, command proceeded;
  22. fail-open: binary removed after init → `rm` still works (re-assert 20's matrix at product level);
  23. gc: retention `max_ops=3` → 5 ops → auto-gc leaves ≥3 (guard-tolerant) and `gc` converges to 3;
  24. purge --all → clean store, doctor ok;
  25. disable/enable + `UNDO_DISABLE=1` paths;
  26. doctor on the sandbox: exit 0;
  27. NOMATCH probe (U3, observational — assert shell healthy + no crash, record op-presence outcome);
  28. timing guard: scenario-1 hook overhead measured (loose < 500 ms CI bound; log the p50 for §14 tracking).
- CI job `e2e` (extends 02): both macOS runners, release build, scenario suite; artifacts on failure: sandbox tree listing + store manifests + captured stderr.

## Detailed Requirements

1. Each scenario is independently runnable (`cargo test e2e::s01_rm_file`) and isolated (own sandbox; no shared state).
2. Assertions use content hashes (SHA-256 via a tiny local helper — no new deps beyond allowlist; `sha2`? NOT allowlisted → use `openssl dgst` shell-out or add `sha2` to ADR-003 allowlist via the documented amendment process — choose `sha2` amendment, it's pure-Rust and dev-scope; record in PR).
3. Sudo scenario must degrade to `skipped` (not failed) when passwordless sudo is unavailable locally.
4. Flaky-tolerance: zero retries permitted; any nondeterminism found must be fixed at the source (deadline margins, sleeps only via store-time backdating helpers from 27's test kit).
5. The matrix table above is mirrored in a `tests/e2e/MATRIX.md` checklist kept in sync (a test enumerates scenario ids vs the file).

## Acceptance Criteria

- [ ] All 28 scenarios green on `macos-15` and `macos-26` release-profile CI.
- [ ] MATRIX.md in sync (enforced by test).
- [ ] Failure artifacts demonstrably useful (force one failure in a scratch commit; screenshot in PR; revert).

## Validation

CI run links (both runners) in the implementation PR.

## Dependencies

20, 21, 22, 26, 27, 28.

## Non-goals

Performance benchmarking beyond loose guards (§14 tracking only), non-macOS runners.

## Design References

DESIGN.md §15, §13 (F1–F20 mapping), §14, §6.4; ISSUE_PLAN §6.
