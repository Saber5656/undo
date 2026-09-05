# undo — v1 Design Specification

Automatic pre-execution snapshots for destructive shell operations, with an `undo` command to roll them back.

- Status: v1 baseline, approved scope (2026-07-07)
- Audience: implementation agents executing [docs/ISSUE_PLAN.md](ISSUE_PLAN.md). This document is the **canonical source of truth**; issues reference sections here by number.
- Decisions: [ADR-001](decisions/ADR-001-interception-model.md) interception, [ADR-002](decisions/ADR-002-snapshot-backend.md) snapshots, [ADR-003](decisions/ADR-003-rust-implementation.md) Rust, [ADR-004](decisions/ADR-004-store-format.md) store, [ADR-005](decisions/ADR-005-fail-open-posture.md) fail-open, [ADR-006](decisions/ADR-006-store-security-model.md) security.
- Research: [prior-art](research/prior-art.md), [macos-platform-notes](research/macos-platform-notes.md).

---

## 1. Product overview

`undo` is a macOS/zsh safety net. A zsh `preexec` hook observes each interactive command line; when the command would destroy file data or metadata (`rm`, `mv`/`cp` overwrites, `sed -i`, `chmod`, `chown`), the core binary snapshots the affected paths **before execution** using APFS clones, then the command runs completely unmodified. Typing `undo` restores the most recent operation; `undo list` / `undo <op-id>` restore older ones.

Differentiation vs. prior art (see research/prior-art.md): every existing tool either replaces the destructive command (trash-cli, rip, safe-rm — habit change, altered semantics, `rm`-only) or works post-hoc without data (oops-cli). `undo` is an **observer**: no habit change, no semantic change, multi-command coverage, restore with real data.

### 1.1 Approved product decisions (2026-07-07)

| Decision | Choice |
|---|---|
| Platform / shell | macOS 13+ (APFS), zsh ≥ 5.8, interactive shells only |
| Protection scope v1 | `rm`, `mv` (moves + overwrites), `cp` (overwrites), `sed -i` / `perl -i`, `chmod`, `chown` (+ their Homebrew `g`-prefixed GNU variants) |
| Implementation | Rust, single binary `undo` |
| Distribution | GitHub Releases (universal2 binary) + Homebrew tap |

## 2. Goals and non-goals

### 2.1 v1 goals

- G1. Zero perceptible overhead on non-destructive commands (< 0.5 ms prefilter, no process spawn).
- G2. Snapshot-before-execution for the protected command set, same-volume cost ≈ metadata-only (APFS clones).
- G3. One-command restore with conflict safety; every restore is itself undoable (inverse operations).
- G4. Fail-open everywhere: the tool must never break, block, or meaningfully delay the user's shell (ADR-005).
- G5. Honest coverage reporting: anything not protectable produces a visible `not protected` notice (§6.4).
- G6. Bounded resource usage: retention caps + GC + purge (ADR-006).
- G7. Secure by default: 0700 store, no network, no telemetry, TM/Spotlight exclusion, no-eval parser (§12).

### 2.2 v1 non-goals (explicit)

- Non-interactive shells, scripts, `xargs`/`find -delete`, subshell-substituted arguments (`$(…)`), variable arguments (`$f`) — warn-not-protected only.
- Shell redirection truncation (`> file`), `git` destructive ops — deferred to v2 (product decision).
- bash / fish / Linux support.
- `sudo`-prefixed commands (privilege boundary).
- Blocking/strict mode; encryption at rest; daemon/watcher mode.

## 3. UX and CLI reference

### 3.1 Install & activate

```sh
brew install <owner>/tap/undo        # or download from GitHub Releases
echo 'eval "$(undo init zsh)"' >> ~/.zshrc
exec zsh
undo doctor                          # verify environment
```

### 3.2 Command grammar

```
undo                       Restore the most recent operation (interactive confirm)
undo <OP_ID>               Restore a specific operation (ULID, unique prefix accepted)
undo list [-n N] [--json]  List recent operations (default N=20)
undo show <OP_ID> [--json] Show one operation in detail (entries, sizes, notices)
undo init zsh              Print the zsh hook script (for eval in .zshrc)
undo doctor [--json]       Environment/health checks (§3.6)
undo gc [--dry-run]        Enforce retention now; report what was/would be evicted
undo purge (<OP_ID> | --all) [--yes]   Delete snapshot data immediately
undo enable | disable      Persistent on/off switch (config-backed)
undo status                Hook active? store stats, retention usage, version
undo config (show | path)  Print effective config / config file path
undo __hook …              Internal: called by the zsh hook (hidden from help)
undo __session-id          Internal: print a fresh ULID (hook uses it for UNDO_SESSION)
```

Global flags: `--store <path>` (override store root; also `UNDO_STORE`), `--config <path>` (also `UNDO_CONFIG`), `--json` where noted, `-y/--yes` (skip confirmation), `--force`, `--dry-run`, `--into <dir>` (restore variants, §9).
Restore-specific flags apply to bare `undo` and `undo <OP_ID>`: `[-y] [--force] [--dry-run] [--into <dir>]`.

### 3.3 Exit codes (all commands)

| Code | Meaning |
|---|---|
| 0 | Success |
| 1 | Runtime error (I/O, unexpected) |
| 2 | Usage error (clap) |
| 3 | Restore aborted: conflicts detected, no changes made (§9.4) |
| 4 | Restore partially failed (some entries restored, some not) |
| 5 | Store integrity/permission/version error (ADR-004, ADR-006) |

`undo __hook` always exits 0 (ADR-005), except under the test-only flag `--strict-errors` (hidden; used by integration tests to assert internal failures).

### 3.4 Output conventions

- Human output: plain text tables, no color dependency; bold/dim via raw ANSI only when stdout is a TTY and `NO_COLOR` is unset.
- `--json`: stable, versioned shapes (`{"format":1,…}`); documented per command in the issue specs; tests consume these.
- All notices/warnings go to **stderr**, prefixed `undo: ` (hook-originated: `undo: not protected: <reason>`).
- Paths echoed to the terminal are escaped for control characters (§12.5).

### 3.5 Example session

```
$ rm -rf build/
$ undo
undo: restore op 01JZK3… ?  rm -rf build/   (12s ago, 1 entry: dir build/ 3,214 files, 182 MB)
Proceed? [y/N] y
undo: restored 1/1 entries → build/
$ undo list
ID          AGE    COMMAND                 ENTRIES  SIZE
01JZK4…     5s     restore 01JZK3…         1        182 MB   (inverse of 01JZK3…)
01JZK3…     40s    rm -rf build/           1        182 MB
```

### 3.6 `undo doctor` checks

| Check | Pass condition | Failure handling |
|---|---|---|
| binary on PATH & not shadowed | `which -a undo` resolves to this binary first | warn + show shadowing entry |
| hook installed | `.zshrc` contains `undo init zsh` | print install line |
| hook active in this shell | `UNDO_SESSION` env set | explain eval/exec zsh |
| zsh version | ≥ 5.8 | fail |
| store perms/owner | 0700, owned by user | fail + suggested `chmod`/`chown` (never auto-fix) |
| store volume clone support | `getattrlist` → `VOL_CAP_INT_CLONE` | warn: copy fallback will be used |
| TM exclusion xattr on store | present | warn + re-apply hint |
| disk free space | > 1 GiB free on store volume | warn |
| config parses | no errors | fail + first error |
| Full Disk Access hint | readdir `~/Desktop` succeeds | info hint about TCC |

## 4. Architecture overview

```
┌─ interactive zsh ─────────────────────────────┐
│ preexec → __undo_preexec()                    │
│   [prefilter: no protected token → return]    │
│   └─ spawns: undo __hook  (stdin: cmd text)   │
└───────────────┬───────────────────────────────┘
                ▼
┌─ undo (Rust binary) ──────────────────────────┐
│ __hook: parse → analyze → plan → snapshot     │
│ CLI:    list/show/restore/gc/purge/doctor/…   │
│                                               │
│ crates::modules                               │
│  cli        clap surface, exit codes          │
│  config     TOML load/merge/validate    (§11) │
│  report     notices, escaping, exit map (§3.4)│
│  analyze    tokenizer, extraction,            │
│             word resolution, per-cmd analyzers│
│             → SnapshotPlan               (§6) │
│  snapshot   clone/copy/metadata backends,     │
│             orchestrator                 (§7) │
│  store      layout, locks, manifest, index(§8)│
│  restore    planner + executor           (§9) │
│  lifecycle  gc, purge, retention        (§10) │
│  doctor     environment checks         (§3.6) │
└───────────────┬───────────────────────────────┘
                ▼
     ~/.local/share/undo/   (store, ADR-004)
```

Sequence — protected command:

```mermaid
sequenceDiagram
    participant U as user
    participant Z as zsh (preexec)
    participant C as undo __hook
    participant S as store
    U->>Z: rm -rf build
    Z->>Z: prefilter: "rm" ∈ protected tokens
    Z->>C: spawn; stdin = expanded cmd text; args: --cwd --session
    C->>C: tokenize → simple cmds → analyzer(rm) → plan {build/: Delete}
    C->>S: stage payload clones + manifest → rename commit
    C-->>Z: exit 0 (always); notices on stderr
    Z->>U: executes rm -rf build (unmodified)
```

## 5. Shell integration contract

### 5.1 `undo init zsh` output

A **static** zsh script (embedded in the binary as an asset; no runtime interpolation of user data — §12.2). Responsibilities:

1. Guard: `[[ -o interactive ]] || return`; do nothing if already loaded (`typeset -g __UNDO_LOADED`).
2. Resolve the binary once (`command -v undo`); if missing, print a one-time warning and self-disable.
3. Export `UNDO_SESSION` (a ULID) if unset.
4. Register `__undo_preexec` via `add-zsh-hook preexec` (autoload `add-zsh-hook`; never clobber user `preexec`).
5. `__undo_preexec`:
   - Return immediately if `UNDO_DISABLE=1` or self-disabled.
   - `local cmd=${3:-$1}` (see research: `$3` = full alias-expanded text).
   - **Prefilter** (pure zsh, no spawn): test `cmd` against a static word-boundary pattern of protected head tokens (`rm|mv|cp|sed|perl|chmod|chown|grm|gmv|gcp|gsed|gchmod|gchown`), matching also after `/` (path-invoked) and after separators (`;`, `&&`, `|`, `(` …). False positives are fine (the core re-checks precisely); false negatives are not (pattern must be a superset).
   - On match: `print -r -- "$cmd" | command undo __hook --cwd "$PWD" --session "$UNDO_SESSION" || true` — stderr passes through to the terminal; stdout discarded.

### 5.2 Hook ↔ core protocol

| Channel | Content |
|---|---|
| stdin | Raw command text, UTF-8, may be multiline; max 1 MiB (larger → core truncates and treats trailing part as unresolvable, warns) |
| argv | `--cwd <abs path>` `--session <ulid>` `--hook-version <N>` |
| exit code | Always 0 (ADR-005) |
| stderr | User-facing notices only (§3.4) |

`UNDO_HOOK_VERSION` constant (integer, starts at 1) is baked into the emitted script and passed via `--hook-version`; on mismatch with the binary's expected version, the core proceeds but emits a warn-level notice (each affected invocation; the notice sink's per-process dedup applies) telling the user to re-`eval` the hook (binary upgraded under a live shell).

### 5.3 Environment variables

| Var | Direction | Meaning |
|---|---|---|
| `UNDO_SESSION` | hook → core | Session ULID, recorded in manifests |
| `UNDO_DISABLE` | user → hook | `1` disables interception for this shell |
| `UNDO_STORE` / `UNDO_CONFIG` | user → core | Path overrides (highest precedence, §11) |
| `NO_COLOR` | user → core | Suppress ANSI styling |

### 5.4 Fail-open requirements (normative)

- Hook must tolerate: missing binary, non-executable binary, core crash, core hang (core self-limits via soft deadline; the hook itself never `wait`s with a timeout mechanism — the core's deadline is the guarantee), read-only store, ENOSPC. In all cases the user's command executes.
- The hook must not change: exit status visible to the user, `$?`, shell options, traps, or any zsh global except its own `__UNDO_*` / `UNDO_SESSION` names.

## 6. Command analysis specification

Input: command text + cwd. Output: per-simple-command `SnapshotPlan` + notices. **The analyzer never executes, evals, or globs-via-shell any input** (§12.1).

### 6.1 Stage 1 — tokenizer (`analyze::lex`)

A conservative POSIX-flavored lexer sufficient for zsh command lines:

- Whitespace-separated words; single quotes (no escapes inside), double quotes (`\"` and `\\` escapes; `$` inside marks the word *unresolvable*, not an error), backslash escapes outside quotes.
- Operator tokens: `;` `&&` `||` `|` `|&` `&` `(` `)` `{` `}` newline.
- Redirection tokens (recognized to be *excluded from operands*): `<` `>` `>>` `&>` `>&` `2>` `2>>` `<<<` and fd-prefixed forms. Heredoc `<<` marks the rest of that simple command unresolvable-conservative (heredoc bodies are not parsed).
- Unresolvable-word markers (word flagged, never expanded, never evaluated): contains unquoted or double-quoted `$` (parameter/command substitution), backtick, `$(`, `${`, process substitution `<(` `>(` `=(`, history `!` designators are treated as literal (preexec receives post-history-expansion text).
- Output: `Vec<Token>` where `Token = Word { text, flags: {quoted, has_glob, unresolvable} } | Op(kind)`.
- On any lexer panic-class input (deeply nested, > 1 MiB, invalid UTF-8 → lossy-converted): emit `ParseOutcome::Unsupported` — the whole line is skipped with a `not protected: could not parse command line` notice. Never crash (fuzz target, issue 29).

### 6.2 Stage 2 — simple-command extraction (`analyze::extract`)

- Split token stream into simple commands at `;`, `&&`, `||`, `|`, `|&`, `&`, `(`, `)`, `{`, `}`, newline. Each pipeline member is analyzed independently.
- For each simple command, iteratively strip **prefixes**:
  - Environment assignments `NAME=value` (POSIX name chars before `=`).
  - Wrapper words: `command`, `builtin`, `exec`, `nocorrect`, `time`; `noglob` (sets `globbing_disabled` for the command); `env` (also strips its `NAME=value` args and its own `-i`/`-u X` flags).
  - `sudo` / `doas` → mark command `Sudo`; analysis stops; notice `not protected: sudo commands are out of scope` (once per line). (ADR-001)
  - `xargs` as head → mark `Unresolvable` (stdin-driven); notice.
- **Head resolution**: first remaining word; if it contains `/`, take the basename. Look up in the protected-command table (config-filterable, §11):

| Head names | Analyzer |
|---|---|
| `rm`, `grm` | rm (§6.5) |
| `mv`, `gmv` | mv (§6.6) |
| `cp`, `gcp` | cp (§6.6) |
| `sed`, `gsed` | in-place (§6.7, BSD/GNU grammar by name) |
| `perl` | in-place (§6.7) |
| `chmod`, `gchmod`, `chown`, `gchown` | metadata (§6.8) |

- Heads that are unresolvable words, or not in the table → no plan for that simple command.

### 6.3 Stage 3 — word resolution (`analyze::resolve`)

Applied to candidate operand words of a matched command:

1. Word `unresolvable` → excluded from the plan; sets the command's `partial = true`.
2. **Tilde expansion**: leading unquoted `~` → `$HOME`; `~user` → `getpwnam(user)` home; unknown user → unresolvable.
3. **Glob expansion** (skipped when `noglob` or word had quotes around meta chars): patterns containing `*`, `?`, `[…]`, or `**/`. Expand with the `glob` crate against `cwd`, options matching zsh defaults: case-sensitive, `*` does not match leading dots (`require_literal_leading_dot`), path separators literal. Results lexicographically sorted.
   - Pattern matching **zero** paths: contributes nothing; other operands still processed (conservative-protective; zsh itself may abort the whole line via NOMATCH — harmless spurious snapshot, §13).
4. **Absolutization**: join with `cwd`, normalize `.` and `..` **lexically**; do NOT resolve symlinks in the final component (analysis is `lstat`-based; symlink operands are captured as links, §7.3). Intermediate-component symlinks are left as the kernel resolves them (we snapshot what `lstat(path)` sees).
5. Existence check (`lstat`): non-existent paths drop out of the plan silently (e.g. `rm` of a typo path).

### 6.4 Not-protected policy (normative list)

These produce a stderr notice (`undo: not protected: …`, level `warn`) and no/partial snapshot:

| Case | Notice reason |
|---|---|
| Unresolvable words in operands (`$var`, `$(…)`, backticks, `${…}`, heredoc) | `argument not resolvable before execution` (marks `partial`) |
| `sudo`/`doas` prefix | `sudo commands are out of scope` |
| `xargs` head / stdin-driven | `stdin-driven arguments` |
| Entry cap / size cap / deadline exceeded | `too large: <cap detail>` (§7.5) |
| Store unavailable (perms, ENOSPC, missing after retry-create) | `store unavailable: <errno>` |
| Line unparseable / > 1 MiB | `could not parse command line` |

Verbose-only notices (`ui.notices = verbose`): excluded-path skips, non-existent operands, protected-command-with-empty-plan.

### 6.5 rm analyzer

- Flags (BSD rm + GNU long): boolean `-d -f -i -I -P -R -r -v -W -x --force --recursive …`; `--` ends flags; unknown `-…` words before `--` are treated as flags (conservative: rm has no valued flags on either platform).
- Every remaining operand (post-resolution) that exists → `PlanEntry { path, disposition: Delete, payload: Content }`. Directories require `-r/-R/-d` for rm to act, but we snapshot regardless of flag presence (rm may still fail; harmless).
- `-i`/`-I` do not suppress snapshots (the user may confirm).

### 6.6 mv / cp analyzers

Shared operand shape: `cmd [flags] SRC… DEST`.

- Valued flags: none on BSD `mv`; `cp` BSD has none valued either (`-X`, `-c`, `-p`, `-R`, `-n`, `-i`, `-f`, `-a`, `-v`, `-L/-P/-H`). GNU variants' valued flags (`--target-directory=…`, `-t DIR`, `--suffix=…`, `-S`) are parsed for `gmv`/`gcp` (`-t DIR` sets DEST explicitly).
- Resolve DEST: if existing directory (stat, follow symlink) → *into-dir* mode: for each SRC, target = `DEST/basename(SRC)`; else pairwise (`SRC DEST`, only valid with a single SRC — multiple SRC with non-dir DEST is a command error; plan only what is coherent).
- `mv` entries:
  - Each existing SRC → `PlanEntry { path: SRC, disposition: Move { dest: TARGET }, payload: MetadataOnly }` (content is preserved by the move itself; undo = move back).
  - Each TARGET that exists (lstat) and `-n` absent → `PlanEntry { path: TARGET, disposition: Overwrite, payload: Content }`.
- `cp` entries: overwrites only — each existing TARGET (single-file mode) → `Overwrite/Content`. For `cp -R SRC DIR`: walk SRC (bounded by caps, §7.5); for each source file whose corresponding `DIR/…` path exists (lstat) → `Overwrite/Content` entry. `-n` → no entries.
- Dedup rule (all analyzers): same path appearing twice keeps the strongest disposition: `Delete > Overwrite > Move > Metadata`.

### 6.7 In-place editor analyzer (`sed -i` / `perl -i`)

Only produces a plan when in-place mode is detected; otherwise no entries.

- `sed` (BSD grammar, head `sed`): `-i` takes a **separate** suffix argument (`-i ''` common). Detection: `-i` present → next word is the suffix (may be empty), consumed. Combined `-iSUFFIX` also accepted (BSD allows attached). Script determination: first non-flag word unless `-e`/`-f` given (then all `-e SCRIPT`/`-f FILE` consume their values and every non-flag word is a file operand). Valued flags: `-e -f -i`; boolean: `-a -E -g -l -n -r -s -u`.
- `gsed` (GNU grammar): `-i[SUFFIX]` attached-only; separate next word is a **file**, not a suffix. Long forms `--in-place[=SUFFIX]`, `-e`, `--expression=…`, `-f`, `--file=…`.
- `perl`: in-place iff `-i[extension]` (attached) or clustered (e.g. `-pi -e …`). Script args: every `-e/-E CODE` consumes a value; if no `-e/-E`, the **first** non-flag operand is the program file (not modified, excluded); remaining operands are files. Valued flags: `-e -E -F -l? (no)` — implement the documented set: `-e -E` valued; `-i` optionally-attached; boolean cluster support (`-pie` style split).
- Each resulting existing file operand → `Overwrite/Content`.
- Any ambiguity (e.g. suffix-vs-file uncertainty) resolves to **snapshotting more rather than less**, but must never mis-consume `--`-terminated operand lists. Grammar table tests are mandatory (issue 17).

### 6.8 chmod / chown analyzer

- First non-flag operand = mode/owner spec (not validated further); remaining operands = paths.
- Boolean flags: `-f -h -v -R` (+ BSD `-H -L -P`); `chown` additionally never has valued flags on BSD; `gchmod/gchown` long forms `--reference=RFILE` marks command unresolvable-partial (we can't know RFILE semantics cheaply — snapshot paths anyway with Metadata disposition; reference file itself is not an operand).
- Non-`-R`: each existing path → `PlanEntry { path, disposition: Metadata, payload: MetadataOnly }`.
- `-R`: recursive walk of each directory operand (caps §7.5), one `Metadata` entry per visited node. `-h` respected implicitly: we record `lstat` of each node (symlinks recorded as themselves).

## 7. Snapshot engine

### 7.1 Inputs/outputs

Input: `SnapshotPlan { entries, command_text, cwd, session }`. Output: a committed operation directory (§8) or a warn-and-proceed abort (ADR-005). Empty plans produce **no** operation.

**One command line = at most one operation.** When a line contains several matched simple commands (`rm a; chmod 600 b`), their per-command results are merged into a single plan, deduplicated across commands with the disposition-strength rule (§6.6), before execution. `undo` therefore rolls back the whole line as one unit.

### 7.2 Backend selection (per entry, per file)

```
if payload == MetadataOnly → record lstat attrs only
else if file's st_dev == store's st_dev → clonefile(2) with CLONE_NOFOLLOW
else (EXDEV) or clonefile error ENOTSUP/EACCES… → bounded byte copy (§7.4)
```

`st_dev` is taken from `lstat` of each file and of the store root (cached).

### 7.3 Directory and special-file handling (per ADR-002)

- Directory entries: engine walks the tree itself (depth-first, no symlink following, `openat`-anchored to resist swaps — §12.4): recreate directories (`mkdir` in payload, record mode/uid/gid), clone/copy regular files, recreate symlinks via `symlink()` with the identical target string (never resolved).
- Hard links within a tree are captured as independent file clones (link identity not preserved — documented in §13).
- Sockets, FIFOs, device nodes: recorded in the manifest (`kind`) with `payload: null`, `not_protected_reason: "special-file"`; restore recreates nothing for them.
- File clones preserve xattrs/ACLs automatically; the copy fallback copies mode + mtime and xattrs via `copyfile(3)` `COPYFILE_XATTR` (no ACLs — noted §13).

### 7.4 Copy fallback bounds

- Per-operation byte budget across all copied (non-cloned) payloads: `limits.max_copy_bytes` (default 512 MiB). Exceeding it aborts the *remaining* copy-entries: they are recorded with `not_protected_reason: "over_size_cap"`, warning emitted; already-copied entries stay.
- Copy is streamed with the budget checked per chunk; a partially-copied payload over budget is deleted.

### 7.5 Global caps and deadline (applies to analysis walks + snapshot execution)

| Limit | Default | Behavior on exceed |
|---|---|---|
| `limits.max_entries_per_op` | 50,000 | Abort plan/walk; op recorded as aborted with warning `too large: entry cap` if nothing useful captured, else partial |
| `limits.max_copy_bytes` | 512 MiB | §7.4 |
| `limits.snapshot_soft_deadline_ms` | 2000 | Clock checked between entries; on exceed, stop, keep already-captured entries, mark op partial, warn |

### 7.6 Commit protocol (with §8)

1. `mkdir <store>/ops.noindex/<ULID>.staging/`
2. Write payloads under `payload/`, indexed by entry number.
3. Capture attrs (`lstat`) into manifest entries.
4. Write `manifest.json` (+ `fsync` file and staging dir).
5. `rename` staging → `<ULID>/` (commit point).
6. Append line to `index.jsonl` under short exclusive `flock` (best-effort; failure ≠ op failure).
7. Trigger auto-GC check (cheap, §10.3).

Crash at any point < 5 leaves only staging garbage (GC-swept). The engine never modifies user files.

## 8. Store layout and manifest schema

### 8.1 Layout (ADR-004)

```
${XDG_DATA_HOME:-~/.local/share}/undo/
  format-version          # "1\n"
  config-note.txt         # pointer to config path (informational)
  lock                    # flock target for GC/restore/index (0600)
  index.jsonl             # rebuildable cache: one JSON line per op
  ops.noindex/
    01JZK3ABCDEF….staging/ # uncommitted (crash leftovers; GC sweeps > 1h old)
    01JZK3ABCDEF…/
      manifest.json
      payload/
        0                 # file clone/copy, or directory tree
        3/…               # (indices are entry indices; gaps = no-payload entries)
```

- Store init: create tree `0700`/files `0600`, write `format-version`, set Time Machine exclusion on the store root, verify owner == euid; wrong owner/perms on later opens → exit 5 with remediation text (never auto-fix). (ADR-006)
- `format-version` newer major than binary supports → exit 5 `store requires a newer undo`.

### 8.2 `manifest.json` schema (format 1)

```jsonc
{
  "format": 1,
  "id": "01JZK3QG4Y9Z8XWVUTSRQPNMKJ",     // ULID, equals dir name
  "created_at": "2026-07-07T03:21:45.123Z", // UTC RFC3339 ms
  "session": "01JZK2…",                     // UNDO_SESSION
  "cwd": "/Users/alice/proj",
  "command": "rm -rf build",                // exact text received (§12.6)
  "tool_version": "0.1.0",
  "hook_version": 1,
  "kind": "command" | "restore",
  "restores": null | "01JZK1…",             // for kind=restore: op restored
  "aborted": false,                          // deadline/cap abort with nothing useful
  "partial": false,                          // some entries not protected
  "logical_bytes": 190840321,                // sum of entry sizes (content payloads)
  "entries": [
    {
      "index": 0,
      "path": "/Users/alice/proj/build",     // absolute, validated (§12.5)
      "kind": "file" | "dir" | "symlink" | "special",
      "disposition": "delete" | "overwrite" | "move" | "metadata",
      "move_dest": null | "/abs/path",       // disposition=move
      "payload": null | "payload/0",         // store-relative
      "mode": 16877, "uid": 501, "gid": 20,  // from lstat
      "mtime": "2026-07-07T03:20:01Z",
      "size_bytes": 190840321,               // files: st_size; dirs: sum
      "entry_count": 3214,                   // dirs: nodes captured
      "not_protected_reason": null | "over_size_cap" | "special-file" | "walk-error:<errno>"
    }
  ],
  "notices": ["…"]                           // notices shown at snapshot time
}
```

Validation on read (strict): `format` known, `id` matches ULID alphabet and dir name, every `path`/`move_dest` absolute with no `..` component, `payload` matches `^payload/[0-9]+$`. Invalid manifest → operation reported as corrupt in `list` (flagged row), refuses restore (exit 5). (§12.5)

### 8.3 `index.jsonl` line schema

```json
{"id":"01JZK3…","at":"2026-07-07T03:21:45Z","kind":"command","cmd":"rm -rf build","entries":1,"logical_bytes":190840321,"partial":false}
```

Cache only: rebuilt by full scan when missing, unreadable, or when a listed id lacks a directory (self-heal, issue 27). Readers must tolerate trailing partial lines.

### 8.4 Locking summary

| Actor | Lock on `<store>/lock` |
|---|---|
| Snapshot (`__hook`) | none for op dir; **short EX** for index append |
| `list`/`show` | none (read-only; tolerate staging/missing) |
| Restore | **SH** for the whole run (blocks GC), plus its own snapshot's index append EX |
| GC / purge | **EX** |

Lock acquisition is non-blocking with a 2 s retry budget; failure → warn (snapshot: skip index append; gc/restore: abort with exit 1/5 messaging).

## 9. Restore engine

### 9.1 State machine

```
LOAD → VALIDATE → PLAN → [conflicts && !--force → ABORT(exit 3)]
     → CONFIRM (TTY prompt unless --yes; --dry-run stops here printing the plan)
     → SNAPSHOT-INVERSE (capture current state of paths restore will touch)
     → EXECUTE (ordered) → RECORD (inverse op manifest, kind=restore)
     → REPORT (per-entry results; exit 0 | 4)
```

### 9.2 Target selection

- Bare `undo`: most recent op with `kind=command` **or** `kind=restore` (restores are undoable too), skipping corrupt/aborted-empty ops.
- `undo <OP_ID>`: full ULID or unique prefix (≥ 4 chars); ambiguous prefix → error listing candidates (exit 2).

### 9.3 Per-entry plan and conflict taxonomy

Processing order within an op (fixed): **1) Move-back, 2) Overwrite-restore, 3) Delete-restore, 4) Metadata-restore.** Within a class: entry index order (dirs were recorded before their contents by the walk; restore recreates parents first).

| Disposition | Restore action | Conflict (default: abort all, exit 3) | With `--force` |
|---|---|---|---|
| move | `rename(move_dest → path)` | `move_dest` missing; or `path` occupied | occupied `path`: inverse-snapshot then replace; missing `move_dest`: entry fails (reported) |
| overwrite | copy/clone payload back over `path` | none (current content is inverse-snapshotted first, so always safe) | n/a |
| delete | materialize payload at `path` | `path` already exists | inverse-snapshot existing, then replace |
| metadata | `chmod`/`chown`/(uid,gid) to recorded values | `path` missing → per-entry skip (reported, not a global abort) | same |

- `chown` restore to a different uid fails without privileges → per-entry failure with clear message (never attempts sudo). Exit 4 if any entry failed.
- `--into <DIR>`: materialize content payloads under `DIR/<entry-index>-<basename>`; no inverse op, no conflicts possible, metadata applied as mode only. Valid for any op.
- Restore writes go through safe-write paths (§12.4): parent dir opened with symlink-refusing traversal; final `rename` into place from a temp name in the same parent.

### 9.4 Inverse operation (undo-of-undo)

Before EXECUTE, the engine builds a synthetic plan covering every path the restore will create/replace/re-attribute, snapshots it as a normal operation with `kind: "restore"`, `restores: <op-id>`, `command: "undo <op-id>"`. Consequently `undo` immediately after a restore rolls it back — no separate redo concept in v1.

## 10. Retention, GC, purge

### 10.1 Retention policy (config §11)

| Key | Default | Meaning |
|---|---|---|
| `retention.max_age_days` | 7 | Ops older are evicted |
| `retention.max_ops` | 500 | Keep at most N committed ops |
| `retention.max_total_bytes` | `"2GiB"` | Logical-bytes cap across ops |
| `retention.min_keep_minutes` | 10 | Never evict ops younger than this (except purge) |

Eviction order: oldest first (ULID order). Size-cap eviction never removes the most recent op even if oversized.

### 10.2 `undo gc`

Under EX lock: sweep `.staging` older than 1 h; delete ops violating age → count → size (reading `index.jsonl`, rebuilding it if inconsistent); compact index to match surviving ops. `--dry-run` prints the eviction list without acting. Deletion = rename op dir to `<id>.trash` then remove recursively (a crash mid-delete leaves `.trash`, swept next run).

### 10.3 Auto-GC

After each committed snapshot: cheap check without lock — count ops via index length and newest-vs-oldest ULID timestamps; if any threshold is exceeded, attempt full GC with **non-blocking** EX lock (skip silently if contended). Budget ≤ 250 ms; if exceeded, stop mid-eviction (resume next trigger).

### 10.4 `undo purge`

`purge <OP_ID>` / `purge --all` (requires `--yes` or TTY confirm): immediate removal of payloads + manifests + index lines under EX lock. Documentation states this is deletion, not forensic shredding (ADR-006).

## 11. Configuration reference

Path: `${XDG_CONFIG_HOME:-~/.config}/undo/config.toml`, override `--config`/`UNDO_CONFIG`. Missing file = all defaults. Unknown keys → startup warning (not an error). Values below are the complete v1 key set with defaults:

```toml
[protect]                 # disable interception per command family
rm = true
mv = true
cp = true
sed = true                # in-place only
perl = true               # in-place only
chmod = true
chown = true

[limits]
max_copy_bytes = "512MiB"          # cross-volume copy budget per op
max_entries_per_op = 50000
snapshot_soft_deadline_ms = 2000

[retention]
max_age_days = 7
max_ops = 500
max_total_bytes = "2GiB"
min_keep_minutes = 10

[exclude]
paths = ["~/Library/Caches", "~/.Trash"]   # store root is ALWAYS excluded (not configurable)

[ui]
notices = "warn"          # off | warn | verbose  (§6.4)
color = "auto"            # auto | always | never
```

- Byte sizes: string with `KiB|MiB|GiB` suffix or integer bytes.
- `exclude.paths`: `~` expanded; prefix match on absolute paths.
- Precedence: CLI flag > env var > config file > default.
- `enabled = false` written by `undo disable` (top-level key; `undo enable` sets true / removes).

## 12. Security model

Threat model summary: the tool runs unprivileged, inside the user's account, with no network. The interesting boundaries are the untrusted *command text*, the filesystem (symlink games, hostile filenames), the store contents (copies of sensitive data), and the supply chain. Multi-user attacks assume the OS user boundary holds; an attacker inside the same account already owns the originals (ADR-006).

### 12.1 Parser boundary (untrusted input: command text)

- The tokenizer/analyzers **never** execute, `eval`, glob-through-shell, or expand parameters/substitutions of input text. Unresolvable constructs are flagged, not resolved (§6.1).
- Hard input limits (1 MiB, token count 65,536, nesting depth 64) with graceful `Unsupported` outcome. Fuzzed in CI (issue 29).

### 12.2 Hook script integrity

- `undo init zsh` output is a compile-time constant string: no interpolation of config, env, or paths into emitted code. (A user-controlled store path can therefore never inject into `.zshrc`-eval'd code.)

### 12.3 Process hygiene

- Command text via stdin (never argv → not visible in `ps`) (ADR-006).
- No temp files outside the store; predictable-path attacks avoided by staging inside the 0700 store.

### 12.4 Filesystem safety (snapshot & restore)

- All metadata reads are `lstat`; clones use `CLONE_NOFOLLOW`; directory walks use `openat`/`O_NOFOLLOW`-anchored traversal so a component swapped to a symlink mid-walk yields an error, not an escape.
- Restore refuses to write through symlinked parents: each target's parent chain is opened with `O_NOFOLLOW | O_DIRECTORY` from the nearest recorded ancestor; mismatch → per-entry conflict. Final placement is `rename` from a temp name within the verified parent.
- The store is always excluded from snapshots (self-reference guard) by path-prefix check before planning.

### 12.5 Store integrity (untrusted-ish inputs: op-ids, manifests)

- CLI op-id inputs validated against the ULID alphabet before any path construction (no traversal via `undo ../../…`).
- Manifest reads strictly validated (§8.2); paths must be absolute without `..`; payload refs must match `^payload/[0-9]+$`. Payload files are addressed by index, never by original filename (hostile-filename immunity).
- Terminal output escapes control characters in paths/commands (no escape-sequence injection into the user's terminal).

### 12.6 Data-at-rest posture

Per ADR-006: 0700/0600 + owner verification (refuse, never auto-fix), retention bounds, `purge`, TM exclusion + `ops.noindex`, no encryption in v1 (rationale documented), command text stored with same protections (may contain secrets).

### 12.7 No-network guarantee

No network code paths exist; `cargo-deny` bans network crates; CI fails if one enters the graph (issue 30). No telemetry, no update checks.

### 12.8 Supply chain & release integrity

- `Cargo.lock` committed; dependency allowlist (ADR-003); `cargo audit` + `cargo deny` gates in CI.
- Releases ship `SHA256SUMS` and GitHub build-provenance attestations (issue 33). Code signing / notarization needs a paid Apple Developer ID → **known unknown** (§17); Homebrew installs are not Gatekeeper-quarantined, so the tap path works unsigned.

### 12.9 Abuse cases considered

| Abuse case | Mitigation |
|---|---|
| Malicious filename `\e]0;pwned\a` echoed to terminal | §12.5 escaping |
| Symlink swap during snapshot walk | §12.4 `openat` traversal |
| Symlink planted at restore target parent | §12.4 refusal → conflict |
| Crafted op-id argument | §12.5 ULID validation |
| Tampered manifest (same-user attacker) | Out of trust boundary; strict validation still prevents accidental traversal |
| Hook injection via store/config paths | §12.2 static hook text |
| Secret in command line visible system-wide | §12.3 stdin protocol |
| Deleted secrets resurrected from backups | §12.6 TM exclusion; retention + purge |

## 13. Failure modes and edge cases (normative behaviors)

| # | Situation | Behavior |
|---|---|---|
| F1 | Store volume full (ENOSPC) mid-snapshot | Abort snapshot, clean staging best-effort, warn, command proceeds |
| F2 | Binary missing/crashed | Hook self-disables for session with one-time warning (§5.4) |
| F3 | Crash mid-snapshot | `.staging` orphan; GC sweeps after 1 h |
| F4 | Deadline exceeded on huge tree | Partial op kept, warn (§7.5) |
| F5 | `chown -R` on 10⁶ files | Entry cap → partial + warn |
| F6 | Command line never executes (zsh NOMATCH after we snapshot) | Spurious op in list; GC'd normally; documented |
| F7 | `rm` fails midway (permissions) | Snapshot exists regardless; restore of unchanged files is a no-op-equivalent (delete-restore conflicts on existing paths → default abort tells the user) |
| F8 | Two shells snapshot concurrently | Disjoint ULID dirs; index appends serialized by flock |
| F9 | GC racing restore | Restore holds SH lock; GC (EX) waits/skips |
| F10 | Store deleted mid-session | Recreated on next op (fresh init); prior history gone (warn) |
| F11 | Hook/binary version skew | Warn once per session (§5.2) |
| F12 | Non-APFS home (rare) | Doctor warns; copy fallback with caps everywhere |
| F13 | iCloud "dataless" file operand | clonefile may materialize or fail → fallback copy path; flagged as known unknown (§17) |
| F14 | Case-insensitive APFS | Byte-exact path handling only; no case folding (research §6) |
| F15 | Hard links inside snapshotted tree | Captured as independent contents; link identity not restored (documented) |
| F16 | ACLs on copy-fallback payloads | Not preserved (xattrs are); documented limitation |
| F17 | `mv` across devices (copy+unlink) | Same plan semantics still correct (move-back = move) |
| F18 | Alias `rm='rm -i'` | `$3` arrives expanded → analyzed with `-i`; snapshot unaffected |
| F19 | User-defined function `rm() {…}` | Prefilter matches; analyzer sees `rm` head — snapshot happens even though the function may not delete (harmless false positive) |
| F20 | TCC-protected path (no FDA) | lstat/open EACCES → entry `walk-error`, warn; doctor hints FDA |

## 14. Performance budgets (normative targets, asserted loosely in E2E)

| Path | Budget |
|---|---|
| Prefilter, non-matching command | < 0.5 ms, zero spawns |
| Matching command, small plan (≤ 3 files, clone path) | < 50 ms p50 / < 150 ms p95 added latency |
| `rm -rf` of 10k-entry tree (clone walk) | < 1.5 s typical; bounded by deadline 2 s |
| `undo list` (500 ops) | < 100 ms via index |
| Restore small op | < 200 ms + confirmation I/O |

Binary size target < 5 MiB; zero background processes; zero open FDs between commands (no daemon, ADR-005/ADR-006).

## 15. Validation strategy

| Layer | What | Where |
|---|---|---|
| L0 unit | tokenizer/extraction/resolution/analyzers (table-driven, ≥ 150 cases total), manifest (de)ser + validation, config, caps | each module issue |
| L1 integration (Rust) | store commit protocol, clone/copy backends on real APFS tempdirs, cross-volume via RAM disk (`hdiutil attach -nomount ram://…`), restore engine E2E on fixtures, GC/purge, locking races (spawned processes) | issues 05–10, 24–28 |
| L2 shell | `zsh -ic` sessions with hook loaded: interception, prefilter negatives, fail-open matrix (missing binary, RO store), env toggles | issues 19–20 |
| L3 acceptance | scripted end-to-end scenario matrix (~25 scenarios: every §6.5–6.8 analyzer happy path, conflicts, caps, gc, purge, disable, doctor, restore-of-restore) against a release-profile binary in CI | issue 31 |
| Security gates | fuzz targets (lexer + analyzers) with CI smoke minutes; `cargo audit`/`cargo deny` (network-crate ban); store-perm refusal tests | issues 29, 30 |
| Release gates | universal binary builds, checksums, provenance attestation, formula install test | issues 33, 34 |

CI: GitHub Actions `macos-15` + `macos-26` (arm64); `fmt --check`, `clippy -D warnings`, `cargo test`, L2/L3 jobs. `zsh -ic` preexec firing is verified as the first task of issue 20 (known unknown §17; fallback: `zpty`-driven sessions).

## 16. Deferred to v2 (do not implement in v1)

- Linux, bash, fish adapters; POSIX-portable core split.
- Redirection truncation (`>`), `git reset/clean/checkout` protection.
- Strict/blocking mode; per-command confirmation.
- Pinned operations; encrypted store; shared-machine hardening.
- Daemon/FSEvents mode for script & non-interactive coverage.
- `undo ui` (TUI browser); zsh completions; `undo diff <op>`.
- Per-volume stores; APFS volume-snapshot atomicity for giant trees.
- Signing/notarization pipeline (pending Apple Developer ID decision).
- Localization (Japanese README/messages).

## 17. Known unknowns (may spawn new issues during implementation)

| # | Unknown | Planned probe |
|---|---|---|
| U1 | Does `preexec` fire under `zsh -ic` in CI (no PTY)? | First task of issue 20; fallback `zpty`/`script -q` harness |
| U2 | `clonefile` behavior on iCloud dataless files | Probe in issue 07; acceptable to fall back to copy |
| U3 | Does preexec fire before NOMATCH glob abort (version-dependent)? | Observation test in issue 20; behavior harmless either way (F6) |
| U4 | GH runner TCC state for `~/Desktop` etc. | Issue 31 setup; use `$TMPDIR` fixtures to avoid |
| U5 | `perl -i` clustered-flag corner grammar | Issue 17 table tests against `man perlrun`; conservative fallback allowed |
| U6 | Homebrew tap formula shape for universal binaries (single URL vs per-arch) | Issue 34 research step |
| U7 | Apple Developer ID availability (signing) | Product-owner decision before first release; not v1-blocking via brew |
| U8 | `index.jsonl` contention under heavy parallel shells | Issue 27 race test; fallback = scan-only mode |

## 18. Glossary

| Term | Meaning |
|---|---|
| Operation (op) | One intercepted command's snapshot record (or one restore) |
| Entry | One path within an op (file/dir/symlink/special) |
| Disposition | What the command does to the path: delete / overwrite / move / metadata |
| Payload | Stored content snapshot (clone or copy) for an entry |
| Observer mode | Snapshot-then-let-run; never modify the user's command (ADR-001) |
| Fail-open | On any internal failure, the user's command still runs (ADR-005) |
| Partial op | Op where some entries could not be protected (§6.4) |
| Inverse op | Snapshot taken by a restore so the restore itself can be undone (§9.4) |
