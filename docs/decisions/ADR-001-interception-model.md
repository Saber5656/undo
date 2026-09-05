# ADR-001: Observer-Mode Interception via zsh `preexec`

- Status: Accepted
- Date: 2026-07-07
- Related: DESIGN.md §5–§6, [macos-platform-notes](../research/macos-platform-notes.md), [prior-art](../research/prior-art.md)

## Context

`undo` must know which files a destructive command is about to affect, *before* it runs, in interactive zsh on macOS (v1 scope). Candidate mechanisms:

1. **Shell functions / aliases wrapping each command** (`rm() { … }`)
2. **PATH shims** (a shim directory ahead of `PATH` containing `rm`, `mv`, …)
3. **zsh `preexec` hook that parses the command line and snapshots proactively** (observer mode)
4. **Syscall-level interception** (Endpoint Security framework, dylib interposition)

## Decision

Use **(3): a `preexec` hook** that forwards the alias-expanded command text (`$3`) to the `undo` core binary, which conservatively parses it, resolves affected paths, snapshots them, and then lets the original command run **unmodified**.

## Rationale

- **Zero behavior change.** The user's command executes exactly as typed. Wrappers (1) and shims (2) change resolution order, break `\rm` / `command rm` expectations differently, can fight user-defined aliases, and shims leak into *non-interactive* child processes (build scripts calling `rm` thousands of times — unacceptable blast radius and overhead).
- **Better coverage of invocation spellings.** The parser sees the literal line, so `rm`, `\rm`, `command rm`, and `/bin/rm` are all recognized (basename normalization). Function/alias wrappers miss all but the first.
- **Required anyway by the protection scope.** The approved v1 scope includes `sed -i`, `chmod`, `chown` — argument-pattern analysis of the command line is unavoidable, so the parser must exist; wrappers would be redundant machinery on top.
- **(4) is disproportionate**: ES framework needs entitlements/root and a daemon; dylib interposition is blocked by SIP for platform binaries. Both conflict with "simple, user-space, open-source CLI".

## Consequences

- Coverage is limited to **interactive zsh commands**. Scripts, `xargs rm`, `find -delete`, command substitutions, and variable-carrying arguments are *not resolvable pre-execution* and are handled by an explicit warn-not-protected policy (DESIGN.md §6.4). This is a documented product boundary, not a bug.
- The parser is a **security boundary**: it must never evaluate the command text (no `eval`, no globbing through the shell, no process substitution). See DESIGN.md §12.1.
- A zsh-side prefilter keeps per-prompt overhead near zero for non-matching commands (DESIGN.md §14).
- `sudo`-prefixed commands are deliberately not protected in v1 (privilege boundary; restoring root-owned state as a user would fail or mislead).
