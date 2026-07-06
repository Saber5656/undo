# undo — v1 Issue Plan

- Status: baseline 2026-07-07
- Canonical design: [docs/DESIGN.md](DESIGN.md). Issue drafts: [docs/issues/](issues/). GitHub Issues are derived artifacts of those drafts.

## 1. v1 completion statement

**v1 is complete when all 34 issues below are closed with their Acceptance Criteria and Validation steps green.** At that point the product delivers: a Rust `undo` binary installable via GitHub Releases and a Homebrew tap; zsh `preexec` interception on macOS covering `rm`, `mv`, `cp` overwrites, `sed -i`/`perl -i`, `chmod`, `chown` (per DESIGN §6); APFS-clone snapshots with copy fallback and caps (§7); a crash-safe store (§8); conflict-safe, self-inversible restore (§9); retention/GC/purge (§10); doctor/enable/disable/status/list/show surfaces (§3); the security model of §12 including fuzzed parsers, no-network CI enforcement, and SECURITY.md; and the E2E acceptance matrix (§15) passing in CI on `macos-15` and `macos-26`. No v1 product behavior exists outside this issue list; discovery-driven additions must come from DESIGN §17's known unknowns and be filed as new issues.

## 2. Issue list in recommended execution order

| # | File | Title | Wave |
|---|---|---|---|
| 01 | [01-repo-scaffolding.md](issues/01-repo-scaffolding.md) | Cargo project scaffolding and CLI skeleton | 0 |
| 02 | [02-ci-pipeline.md](issues/02-ci-pipeline.md) | CI pipeline: fmt, clippy, tests on macOS runners | 0 |
| 03 | [03-error-and-notice-foundation.md](issues/03-error-and-notice-foundation.md) | Error taxonomy, exit-code map, notice reporting | 0 |
| 04 | [04-config-module.md](issues/04-config-module.md) | Config module and `undo config` command | 0 |
| 05 | [05-store-layout-and-locking.md](issues/05-store-layout-and-locking.md) | Store layout, init, permission checks, locking | 0 |
| 06 | [06-operation-model-and-manifest.md](issues/06-operation-model-and-manifest.md) | Operation model, manifest schema, index cache | 0 |
| 07 | [07-clonefile-backend.md](issues/07-clonefile-backend.md) | clonefile(2) snapshot backend | 1 |
| 08 | [08-copy-fallback-backend.md](issues/08-copy-fallback-backend.md) | Bounded copy fallback backend | 1 |
| 09 | [09-metadata-recorder.md](issues/09-metadata-recorder.md) | Metadata recorder (lstat capture, recursive walk) | 1 |
| 10 | [10-snapshot-orchestrator.md](issues/10-snapshot-orchestrator.md) | Snapshot orchestrator: plan → committed operation | 1 |
| 11 | [11-command-tokenizer.md](issues/11-command-tokenizer.md) | Command-line tokenizer (no-eval security boundary) | 2 |
| 12 | [12-simple-command-extraction.md](issues/12-simple-command-extraction.md) | Simple-command extraction and head resolution | 2 |
| 13 | [13-word-resolution.md](issues/13-word-resolution.md) | Word resolution: tilde, globs, absolutization | 2 |
| 14 | [14-rm-analyzer.md](issues/14-rm-analyzer.md) | rm analyzer | 2 |
| 15 | [15-mv-analyzer.md](issues/15-mv-analyzer.md) | mv analyzer | 2 |
| 16 | [16-cp-analyzer.md](issues/16-cp-analyzer.md) | cp analyzer | 2 |
| 17 | [17-inplace-editor-analyzer.md](issues/17-inplace-editor-analyzer.md) | sed/perl in-place analyzer | 2 |
| 18 | [18-chmod-chown-analyzer.md](issues/18-chmod-chown-analyzer.md) | chmod/chown analyzer | 2 |
| 19 | [19-hook-entrypoint.md](issues/19-hook-entrypoint.md) | `undo __hook` entry point (analyze→snapshot pipeline) | 3 |
| 20 | [20-zsh-hook-script.md](issues/20-zsh-hook-script.md) | zsh hook script and `undo init zsh` | 3 |
| 21 | [21-doctor-command.md](issues/21-doctor-command.md) | `undo doctor` | 3 |
| 22 | [22-enable-disable-status.md](issues/22-enable-disable-status.md) | enable / disable / status commands | 3 |
| 23 | [23-list-show-commands.md](issues/23-list-show-commands.md) | `undo list` and `undo show` | 4 |
| 24 | [24-restore-planner.md](issues/24-restore-planner.md) | Restore planner and conflict detection | 4 |
| 25 | [25-restore-executor.md](issues/25-restore-executor.md) | Restore executor with inverse operations | 4 |
| 26 | [26-restore-cli.md](issues/26-restore-cli.md) | Restore CLI UX (bare `undo`, `undo <id>`) | 4 |
| 27 | [27-gc-retention.md](issues/27-gc-retention.md) | GC, retention enforcement, index self-heal | 4 |
| 28 | [28-purge-command.md](issues/28-purge-command.md) | `undo purge` | 4 |
| 29 | [29-fuzz-targets.md](issues/29-fuzz-targets.md) | Fuzz targets for parser boundary + CI smoke | 5 |
| 30 | [30-supply-chain-and-security-docs.md](issues/30-supply-chain-and-security-docs.md) | cargo-audit/deny gates, SECURITY.md | 5 |
| 31 | [31-e2e-acceptance-suite.md](issues/31-e2e-acceptance-suite.md) | End-to-end acceptance scenario suite | 5 |
| 32 | [32-readme-and-project-docs.md](issues/32-readme-and-project-docs.md) | README, CONTRIBUTING, LICENSE | 5 |
| 33 | [33-release-workflow.md](issues/33-release-workflow.md) | Release workflow: universal binary, checksums, provenance | 5 |
| 34 | [34-homebrew-tap.md](issues/34-homebrew-tap.md) | Homebrew tap formula and publication handoff | 5 |

## 3. Dependency table

| Issue | Depends on (hard) | Notes |
|---|---|---|
| 01 | — | |
| 02 | 01 | |
| 03 | 01 | |
| 04 | 03 | |
| 05 | 03 | |
| 06 | 04, 05 | manifest+index write/read path |
| 07 | 05 | |
| 08 | 04, 05 | caps from config |
| 09 | 04 | entry caps |
| 10 | 06, 07, 08, 09 | |
| 11 | 03 | |
| 12 | 11 | |
| 13 | 04, 12 | excludes from config |
| 14–18 | 13 | independent of each other; parallelizable |
| 19 | 10, 14–18 | |
| 20 | 19 | includes U1/U3 probes |
| 21 | 05, 04, 20 | checks hook artifacts |
| 22 | 04, 20 | |
| 23 | 06 | parallel with wave-2/3 after 06 |
| 24 | 06 | pure logic vs manifests |
| 25 | 10, 24 | inverse ops reuse orchestrator |
| 26 | 25 | |
| 27 | 06 | EX-lock interplay tested with 25 present |
| 28 | 27 | shares eviction machinery |
| 29 | 14–18 | fuzzes lexer+analyzers |
| 30 | 02 | |
| 31 | 20, 26, 27, 28, 21, 22 | whole-product matrix |
| 32 | 26 (behavioral accuracy), 30 (SECURITY.md exists) | LICENSE needs product-owner confirmation |
| 33 | 02, 31 | release gate = acceptance green |
| 34 | 33 | needs a published release artifact; tap repo creation is a product-owner manual step |

## 4. Implementation waves

| Wave | Issues | Goal | Parallelism |
|---|---|---|---|
| 0 Foundations | 01–06 | Buildable binary, config, store, manifests | 03→04/05 fan-out; 06 joins |
| 1 Snapshot engine | 07–10 | Committed operations from synthetic plans | 07/08/09 parallel |
| 2 Analysis | 11–18 | Command text → SnapshotPlan | 14–18 fully parallel after 13 |
| 3 Hook & environment | 19–22 | Live interception in real zsh | 21/22 parallel after 20 |
| 4 Restore & lifecycle | 23–28 | Restore, GC, purge complete | 23/24 start right after 06 |
| 5 Hardening & release | 29–34 | Security gates, acceptance, docs, distribution | 29/30/32 parallel; 33→34 serial |

## 5. Coverage table — DESIGN.md sections → issues

| DESIGN § | Topic | Covered by |
|---|---|---|
| §3.2–3.4 CLI grammar, exit codes, output conventions | 01, 03, 23, 26, 04, 22 |
| §3.5 example flows | 31 |
| §3.6 doctor | 21 |
| §4 architecture/module layout | 01 |
| §5.1 hook script | 20 |
| §5.2 protocol | 19, 20 |
| §5.3 env vars | 19, 20, 22 |
| §5.4 fail-open | 19, 20, 31 |
| §6.1 tokenizer | 11 |
| §6.2 extraction | 12 |
| §6.3 resolution | 13 |
| §6.4 not-protected policy | 12, 13, 19 (emission), 31 (assertions) |
| §6.5 rm | 14 |
| §6.6 mv/cp | 15, 16 |
| §6.7 in-place | 17 |
| §6.8 chmod/chown | 18 |
| §7.2 backend selection | 10 |
| §7.3 dir/special handling | 07, 09 |
| §7.4 copy bounds | 08 |
| §7.5 caps/deadline | 09, 10 |
| §7.6 commit protocol | 06, 10 |
| §8.1 layout/init | 05 |
| §8.2 manifest schema | 06 |
| §8.3 index | 06, 27 |
| §8.4 locking | 05, 27, 25 |
| §9 restore engine | 24, 25, 26 |
| §10 retention/GC/purge | 27, 28 |
| §11 config | 04 |
| §12.1 parser boundary | 11, 29 |
| §12.2 hook integrity | 20 |
| §12.3 process hygiene | 19 |
| §12.4 fs safety | 07, 09, 25 |
| §12.5 store integrity/escaping | 03, 06, 23, 26 |
| §12.6 data at rest | 05, 28, 30 |
| §12.7 no-network | 30 |
| §12.8 supply chain/release | 30, 33 |
| §12.9 abuse cases | 29, 31 (regression scenarios) |
| §13 failure modes F1–F20 | distributed: F1/F3/F4 → 10; F2 → 20; F5 → 09; F6/F18/F19 → 31; F7–F9 → 25/27; F10–F11 → 19/20; F12 → 21; F13 → 07; F14–F17 → 07/08/24; F20 → 09/21 |
| §14 performance budgets | 20 (prefilter), 31 (loose assertions) |
| §15 validation strategy | 02, 29, 30, 31 |
| §16 v2 deferrals | none (guard: issues' Non-goals) |
| §17 known unknowns | probes embedded in 07, 17, 20, 27, 31, 33, 34 |

Every DESIGN section with implementable behavior maps to at least one issue; issues' Non-goals sections keep §16 out of v1.

## 6. Validation strategy (product level)

Per DESIGN §15. Gates in order: (1) per-issue unit/integration tests green in CI (`macos-15`, `macos-26`); (2) wave-3 exit requires live-shell interception demo (L2) green including the fail-open matrix; (3) wave-4 exit requires restore round-trip property: for every analyzer scenario, `snapshot → destroy → restore` reproduces content+mode byte-exact (L3 subset); (4) v1 exit = full L3 acceptance matrix (~25 scenarios) + fuzz smoke + audit/deny green + release dry-run artifact installs and passes `undo doctor` on a clean runner.

## 7. Deferred v2 items

Tracked in DESIGN §16 (Linux/bash/fish, redirection & git ops, strict mode, pinning, encryption, daemon mode, TUI, per-volume stores, signing pipeline, localization). Do not file v1 issues for these.

## 8. Known unknowns that may create additional issues

DESIGN §17 U1–U8. Rules: an implementer hitting a known unknown documents findings in the affected issue's PR; if resolution changes design behavior, update DESIGN.md first, then file a follow-up issue derived from it. Most likely new-issue sources: U1 (PTY harness for CI), U6 (tap formula shape), U7 (signing pipeline — v2 by default).
