# Wave 10 concrete review-resolution addendum

Repository: Saber5656/undo
Pull request: #35

This addendum is the normative response to the 36 mapped human review findings on
this documentation-only pull request. Each finding has one resolution and one
focused verification gate. The independent review manifest pins the current head,
base, and addendum blob; mutable identities are intentionally omitted here. Any
head or base change invalidates this document and requires fresh review.

The text below is a design/issue contract, not a claim that implementation,
testing, security review, or release validation has already passed. Resolution
requires the focused gate, repository full validation, and applicable
security/privacy acceptance for the pinned identity. No PR review bot is triggered
or rerun.

## Mandatory completion gates

The implementation owner must provide evidence for each focused gate and the
repository's complete validation. Security/privacy review must cover path
traversal, symlink handling, shell parsing, restore writes, and no-network claims
where applicable. Missing, pending, skipped, cancelled, timed-out, stale, or
unknown evidence blocks resolution. A final merge gate must independently
re-fetch current identity, policy, checks, reviews, and unresolved threads.

## Thread contracts

### 1. Emit a notice for explicit out-of-scope commands

- Thread: PRRT_kwDOTNj_oc6OtQOl
- Location: docs/DESIGN.md:251
- Normative resolution: When a simple-command head is not protected or is
  explicitly out of scope, analysis emits one warn-level stderr notice with the
  command head and reason code not-protected, creates no plan for that command,
  and continues the shell line.
- Focused gate: Analyze a protected command, an unknown head, and a sudo-prefixed
  command and assert the exact notice, empty plan, and continued parsing.

### 2. Keep --into undoable

- Thread: PRRT_kwDOTNj_oc6OtQOo
- Location: docs/DESIGN.md:477
- Normative resolution: --into remains a restore target override and the restore
  records a complete inverse operation. The inverse manifest includes every
  created, replaced, and re-attributed target under the --into directory with
  source/destination fields needed to undo that restore.
- Focused gate: Restore a fixture with --into, inspect the recorded restore
  manifest, run bare undo, and assert that only the materialized --into targets
  return to their pre-restore state.

### 3. Split coverage matrix before docs gate breaks

- Thread: PRRT_kwDOTNj_oc6OtQOs
- Location: docs/ISSUE_PLAN.md:143
- Normative resolution: The coverage matrix is represented as separate valid
  Markdown tables for each validation layer, with one header and a matching
  number of cells in every row. No row combines unrelated gates with extra cells.
- Focused gate: Run the repository Markdown lint command and a table parser and
  assert no malformed-row or table-column errors at the coverage matrix.

### 4. Use an output assertion here, not round-trip

- Thread: PRRT_kwDOTNj_oc6OtQOy
- Location: docs/issues/03-error-and-notice-foundation.md:25
- Normative resolution: The test asserts the exact escaped terminal output emitted
  for a control-character path, including the escape representation, rather than
  escaping and then decoding the result in a round-trip assertion.
- Focused gate: Pass a path containing ESC, BEL, and newline and assert the literal
  escaped bytes in captured stderr with no raw control characters.

### 5. Split permission and owner-mismatch cases

- Thread: PRRT_kwDOTNj_oc6OtQO2
- Location: docs/issues/05-store-layout-and-locking.md:35
- Normative resolution: Store validation reports E_STORE_PERMISSION when mode or
  access prevents safe use and E_STORE_OWNER_MISMATCH when ownership differs from
  the invoking user. Each code has its own remediation text and neither case is
  folded into the other.
- Focused gate: Exercise mode-denied and foreign-owner fixtures and assert the
  distinct code, message, and refusal behavior for each.

### 6. Add an explicit recursion bit to SnapshotPlan

- Thread: PRRT_kwDOTNj_oc6OtQO7
- Location: docs/issues/10-snapshot-orchestrator.md:19
- Normative resolution: SnapshotPlan contains recursive: boolean for every planned
  entry. The analyzer sets it from the command semantics and the snapshot engine
  consumes it for the bounded walk; the value is serialized in the committed
  manifest.
- Focused gate: Build non-recursive and recursive plans and assert the field is
  false/true in the plan and committed manifest without inferring it from path
  shape.

### 7. Make the partial/aborted boundary deterministic

- Thread: PRRT_kwDOTNj_oc6OtQPL
- Location: docs/issues/10-snapshot-orchestrator.md:22
- Normative resolution: A plan is partial when at least one entry was captured and
  another entry was skipped or failed. An operation is aborted when zero entries
  were captured because of a fatal or interrupted failure; aborted-empty
  operations are not committed as undoable operations.
- Focused gate: Run zero-entry fatal, one-entry-plus-failure, and all-success
  fixtures and assert aborted, partial, and committed states respectively.

### 8. Align WordToken contract with unquoted_text

- Thread: PRRT_kwDOTNj_oc6OtQPM
- Location: docs/issues/11-command-tokenizer.md:16
- Normative resolution: WordToken exposes unquoted_text containing the token with
  quote syntax and escape syntax removed only where the tokenizer has established
  that removal is safe; the original raw text and quote metadata remain available
  for analysis.
- Focused gate: Tokenize quoted, escaped, and plain words and assert raw,
  unquoted_text, and quote metadata fields against the contract.

### 9. Make unresolvable detection segment-aware

- Thread: PRRT_kwDOTNj_oc6OtQPO
- Location: docs/issues/11-command-tokenizer.md:22
- Normative resolution: The analyzer marks a word unresolvable only when an
  expansion or unsupported construct occurs in an unquoted segment. Literal
  single-quoted text and escaped dollar/backtick characters remain resolvable.
- Focused gate: Analyze literal, escaped, quoted, and unquoted variable/command
  expansion fixtures and assert partial status only for the unquoted expansion
  cases.

### 10. Do not silently drop zero-match globs

- Thread: PRRT_kwDOTNj_oc6OtQPW
- Location: docs/issues/13-word-resolution.md:20
- Normative resolution: A glob that matches zero paths produces an E_NO_MATCH
  finding for the command, contributes no plan entry, and leaves other operands
  subject to normal resolution. The condition is visible in the command result.
- Focused gate: Resolve a command with one unmatched glob and one existing operand
  and assert the finding, the existing operand plan, and no unmatched entry.

### 11. Make exclusion matching path-boundary aware

- Thread: PRRT_kwDOTNj_oc6OtQPb
- Location: docs/issues/13-word-resolution.md:21
- Normative resolution: An exclusion path matches itself and descendants only when
  the next character is a path separator; the canonical component comparison never
  treats foo as a match for foobar.
- Focused gate: Apply exclusions for foo to foo, foo/bar, and foobar and assert only
  the first two are excluded.

### 12. Fix grm typo in GNU rm forms

- Thread: PRRT_kwDOTNj_oc6OtQPe
- Location: docs/issues/14-rm-analyzer.md:16
- Normative resolution: The GNU-compatible command forms table and analyzer use
  rm as the command spelling; grm does not appear as a supported alias in this
  contract.
- Focused gate: Run the analyzer command-name table test and assert rm is accepted
  while the mistaken grm spelling is not introduced by the generated docs.

### 13. Remove -i from BSD sed boolean bucket

- Thread: PRRT_kwDOTNj_oc6OtQPi
- Location: docs/issues/17-inplace-editor-analyzer.md:16
- Normative resolution: BSD sed treats -i as a valued option that consumes the
  next word as its suffix, including an empty suffix. It is excluded from the
  boolean-option set and cannot be interpreted as a standalone boolean.
- Focused gate: Parse sed -i '' file and sed -n file and assert suffix consumption
  only in the first case and file selection only in the second.

### 14. Fix the follow-up issue reference

- Thread: PRRT_kwDOTNj_oc6OtQPk
- Location: docs/issues/18-chmod-chown-analyzer.md:25
- Normative resolution: The cross-command merge behavior reference points to
  issue 10, Snapshot orchestrator, and DESIGN §7.1. Issue 18 returns per-command
  results only; issue 10 owns merging those results into one line operation.
- Focused gate: Validate the issue reference against ISSUE_PLAN and assert the
  document contains issue 10 with the Snapshot orchestrator title and no issue 19
  reference for cross-command merging.

### 15. Make enable write an explicit true value

- Thread: PRRT_kwDOTNj_oc6OtQPn
- Location: docs/issues/22-enable-disable-status.md:24
- Normative resolution: The enable command atomically rewrites the config with
  enabled: true. It parses and validates the existing document before replacing
  it, and a parse or write failure leaves the original bytes unchanged.
- Focused gate: Enable a disabled fixture and inspect true; then use malformed and
  unwritable fixtures and assert refusal with byte-identical originals.

### 16. Add a real failure representation to plan

- Thread: PRRT_kwDOTNj_oc6OtQPr
- Location: docs/issues/24-restore-planner.md:24
- Normative resolution: The restore plan has Action::Failed { code, reason } for
  entries that cannot be safely executed. Failed actions remain in plan order and
  are rendered in the per-entry report; they are not represented as absent actions.
- Focused gate: Plan a missing move source, a permission-denied target, and a
  valid entry and assert two explicit Failed actions plus one executable action.

### 17. Unify inverse-snapshot failure exit code

- Thread: PRRT_kwDOTNj_oc6OtQPx
- Location: docs/issues/25-restore-executor.md:24
- Normative resolution: Any failure to capture the inverse snapshot before restore
  execution returns exit code 4, records the failure code and affected paths, and
  performs no restore writes.
- Focused gate: Inject inverse snapshot failure and assert exit 4, a failure
  report, unchanged targets, and no execute-phase call.

### 18. Add release/tap blockers before promising install docs

- Thread: PRRT_kwDOTNj_oc6OtQP0
- Location: docs/issues/32-readme-and-project-docs.md:17
- Normative resolution: README install documentation is gated on issues 21
  (doctor wording), 26 (restore CLI), 30 (security/release gates), 33 (release
  workflow), and 34 (Homebrew tap). The issue dependency list names all five
  blockers and the README does not claim an install path before they are green.
- Focused gate: Check the dependency graph and assert all five issue numbers are
  present before the README install acceptance gate is enabled.

### 19. Fix install method signature

- Thread: PRRT_kwDOTNj_oc6OtQP1
- Location: docs/issues/34-homebrew-tap.md:19
- Normative resolution: The Homebrew formula defines the install method with valid
  Ruby syntax, def install ... end, and places binary installation inside that
  method.
- Focused gate: Run Ruby syntax validation on the formula and execute the formula
  audit parser without a syntax error.

### 20. Keep index out of Spotlight

- Thread: PRRT_kwDOTNj_oc6OtQZ2
- Location: docs/DESIGN.md:373
- Normative resolution: The entire operation store is rooted at ops.noindex,
  including index.jsonl and every operation directory, so Spotlight cannot index
  either payload content or the index.
- Focused gate: Initialize a store, assert the root basename is ops.noindex, and
  verify index and operation paths are descendants of that root.

### 21. Do not promise warnings where the hook never runs

- Thread: PRRT_kwDOTNj_oc6OtQZ6
- Location: docs/DESIGN.md:41
- Normative resolution: Documentation states that warning notices are emitted only
  when the interactive zsh hook executes; non-interactive shells and unsupported
  shell contexts are documented limitations and do not receive a runtime-warning
  promise.
- Focused gate: Compare the warning contract with the hook entrypoint matrix and
  assert every non-interactive/unsupported case is listed as a limitation.

### 22. Avoid overwriting --into destinations

- Thread: PRRT_kwDOTNj_oc6OtQZ7
- Location: docs/DESIGN.md:472
- Normative resolution: Before any --into write, the executor rejects an existing
  destination or symlink with a conflict result and leaves it unchanged. It never
  replaces an existing --into target.
- Focused gate: Restore into an existing file, directory, and symlink target and
  assert conflict, unchanged bytes, and zero replacement calls.

### 23. Capture move-back sources in inverse ops

- Thread: PRRT_kwDOTNj_oc6OtQZ-
- Location: docs/DESIGN.md:477
- Normative resolution: Every inverse move operation stores both the forward
  move_dest and the original path. Undoing a restore executes move_dest -> path
  after validating both paths against the safe-write rules.
- Focused gate: Restore a moved entry, inspect the inverse manifest for both fields,
  and assert the next undo moves the materialized item back to the original path.

### 24. Cache resolved hook binary path

- Thread: PRRT_kwDOTNj_oc6OtQaA
- Location: docs/DESIGN.md:190
- Normative resolution: init resolves the absolute hook binary path once, validates
  it, and persists it in the configuration. Hook invocations use that stored path
  and do not resolve the binary through PATH.
- Focused gate: Initialize with a known binary, then invoke with a hostile PATH and
  assert the stored absolute path is executed.

### 25. Treat cp -a as recursive

- Thread: PRRT_kwDOTNj_oc6OtQaC
- Location: docs/DESIGN.md:294
- Normative resolution: The cp analyzer maps -a to recursive traversal before
  operand planning, so a directory source under cp -a receives the same bounded
  walk semantics as cp -R.
- Focused gate: Analyze cp -a source destination and assert recursive plan entries
  for nested files plus the configured walk cap.

### 26. Suppress moves skipped by -n

- Thread: PRRT_kwDOTNj_oc6OtQaE
- Location: docs/DESIGN.md:292
- Normative resolution: For mv -n, an existing target produces no Move or
  Overwrite action and the analyzer records a skipped-no-clobber result. A missing
  target retains the normal Move action.
- Focused gate: Analyze existing-target and missing-target mv -n fixtures and assert
  no action in the first case and one Move action in the second.

### 27. Honor recursive symlink traversal flags

- Thread: PRRT_kwDOTNj_oc6OtQaG
- Location: docs/DESIGN.md:312
- Normative resolution: Recursive walks use BSD/GNU semantics: -P never follows
  symlink directories, -H follows symlink directories named on the command line,
  and -L follows symlink directories encountered during the walk. The selected
  mode is recorded in SnapshotPlan.
- Focused gate: Walk command-line and nested symlink fixtures under -P, -H, and -L
  and assert exactly the permitted descendants are planned.

### 28. Guard store through symlinked paths

- Thread: PRRT_kwDOTNj_oc6OtQaK
- Location: docs/DESIGN.md:564
- Normative resolution: Before any store read or write, every ancestor from the
  configured store path to the trusted filesystem root is checked by canonical
  device/inode identity and rejected if it is a symlink or leaves the expected
  ancestry. The operation fails closed without touching the alias target.
- Focused gate: Replace the store parent with a symlink to an outside directory and
  assert initialization/read/write refusal and unchanged outside contents.

### 29. Enforce no-network beyond dependency bans

- Thread: PRRT_kwDOTNj_oc6OtQaN
- Location: docs/DESIGN.md:578
- Normative resolution: The source-level CI gate rejects imports or calls to
  std::net, std::os::unix::net, libc socket APIs, and DNS/network clients, while
  cargo-deny rejects network crates in the dependency graph. A violation fails CI.
- Focused gate: Run the source scanner and cargo-deny check against fixtures that
  add each forbidden import/API and assert a failing result for each.

### 30. Avoid upgrading restore lock for inverse indexing

- Thread: PRRT_kwDOTNj_oc6OtQaS
- Location: docs/DESIGN.md:438
- Normative resolution: Restore holds the shared lock for the restore run and uses
  a separate short exclusive index lock for inverse-operation indexing; it never
  upgrades the shared restore lock to exclusive.
- Focused gate: Instrument lock acquisition during restore and assert one shared
  restore lock plus a distinct short index lock with no shared-to-exclusive upgrade.

### 31. Add hook-side bound for core hangs

- Thread: PRRT_kwDOTNj_oc6OtQaT
- Location: docs/DESIGN.md:214
- Normative resolution: The hook gives each core invocation a 2-second wall-clock
  bound. On timeout it terminates the child, emits the fixed undo-unavailable
  notice, returns success to the user's shell, and records no partial operation.
- Focused gate: Run a core fixture that hangs for 3 seconds and assert hook return
  within 2 seconds, child termination, notice emission, and unchanged shell status.

### 32. Preserve moved content across later commands

- Thread: PRRT_kwDOTNj_oc6OtQaX
- Location: docs/DESIGN.md:320
- Normative resolution: Multi-command planning maintains a sequential virtual
  filesystem state. Each command applies its planned move/delete/overwrite to that
  state before the next command resolves operands, so later commands see moved
  content at its current path.
- Focused gate: Plan mv a b followed by chmod a and assert the second command
  addresses the virtual moved state according to the recorded sequence.

### 33. Snapshot cp destination symlink targets

- Thread: PRRT_kwDOTNj_oc6OtQaZ
- Location: docs/DESIGN.md:294
- Normative resolution: When cp will write through a destination symlink, the
  analyzer records the symlink-resolved target's preimage before the copy and
  records the link path separately for safe restore validation.
- Focused gate: Copy through a destination symlink, inspect the plan for the target
  preimage, and assert restore returns target bytes without replacing the link.

### 34. Do not classify GNU sed script as a file

- Thread: PRRT_kwDOTNj_oc6OtQac
- Location: docs/DESIGN.md:302
- Normative resolution: GNU sed consumes -e/--expression and -f/--file values as
  scripts; the first remaining non-option is a file operand. A script token is
  never added to the snapshot plan.
- Focused gate: Analyze gsed -i -e script file and gsed -i script file fixtures and
  assert the script is excluded while file is the only planned operand.

### 35. Mark shell compound lists unsupported

- Thread: PRRT_kwDOTNj_oc6OtQaf
- Location: docs/DESIGN.md:234
- Normative resolution: The command parser accepts only the documented simple
  command grammar. A compound list it cannot model produces a not-protected
  notice and no inferred snapshot plan; it is never silently treated as a simple
  command.
- Focused gate: Parse pipe, command-substitution, and grouped-list fixtures and
  assert explicit not-protected results with no plan entries.

### 36. Restore cross-device moved directories

- Thread: PRRT_kwDOTNj_oc6OtQai
- Location: docs/issues/25-restore-executor.md:19
- Normative resolution: When moving a directory back with EXDEV, the executor
  recursively copies into a same-filesystem temporary directory, verifies the
  bounded tree, places it with the safe rename path, and removes the source only
  after successful placement. Any failure retains the source and records a
  per-entry failure with exit code 4.
- Focused gate: Force EXDEV for a moved directory containing nested files and
  symlinks and assert byte/metadata restoration, source removal after success, and
  source retention after injected copy failure.

## Resolution boundary

The implementation owner must update the affected design and issue text
consistently, attach evidence to each mapped thread, and resolve only after the
focused, full-validation, and specialist gates pass for the pinned identity.
