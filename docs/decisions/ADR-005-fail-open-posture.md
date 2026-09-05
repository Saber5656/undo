# ADR-005: Fail-Open, Warn-and-Proceed Safety Posture

- Status: Accepted
- Date: 2026-07-07
- Related: DESIGN.md §5.4, §13

## Context

When `undo` cannot protect a command — parser limitation, snapshot failure (disk full, permissions, caps exceeded, deadline), missing binary, corrupted store — should it block the user's command, or let it run unprotected?

## Decision

`undo` is a **best-effort safety net, never a gatekeeper**:

1. The hook and core **never block, modify, cancel, or delay (beyond the soft deadline) the user's command**.
2. Every protection failure degrades to a **one-line warning on stderr** (`undo: not protected: <reason>`), and the command proceeds.
3. The `__hook` entry point **always exits 0** (a non-zero exit could interfere with shell state or user `preexec` chains); failures are communicated via stderr text only.
4. The zsh hook is written so that even a crashing or missing binary cannot break the shell (`command … || true`, existence check at init with one-time self-disable).
5. A "strict mode" that blocks unprotectable commands is explicitly **deferred to v2** and must not be partially implemented in v1.

## Rationale

- The product's value proposition is "invisible until you need it". A protection tool that sometimes breaks or delays normal shell usage will be uninstalled immediately — negative net safety.
- Honest, visible degradation ("not protected: argument contains `$(…)`") teaches users the tool's real coverage boundary instead of creating false confidence.
- Blocking semantics require a completely different UX contract (confirmation prompts inside preexec, TTY handling) and a much higher correctness bar for the parser (false positives become denial-of-service on the user's own shell).

## Consequences

- Warn messages must be rate-limited per reason per session where they could repeat noisily (config `ui.notices = off|warn|verbose`, default `warn`).
- Tests must assert fail-open behavior explicitly: binary removed, store chmod'd to read-only, disk-full simulation, deadline exceeded — in every case the guarded command still runs (issues 19, 20, 31).
- Snapshot soft deadline (`limits.snapshot_soft_deadline_ms`, default 2000): the engine checks the clock between entries and aborts the *snapshot* (never the command) when exceeded, recording a partial/aborted op with a warning.
- The false-negative surface (what we knowingly do not protect) is enumerated in DESIGN.md §6.4 and mirrored in the README so it is a documented contract, not a surprise.
