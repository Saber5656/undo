# Research: Prior Art — Shell Deletion-Protection and Undo Tools

- Date: 2026-07-07
- Status: informs [ADR-001](../decisions/ADR-001-interception-model.md) and DESIGN.md §1–2
- Method: web survey of existing tools and their protection models

## Landscape

| Tool | Model | Coverage | Restore | Key limitation vs. our goal |
|---|---|---|---|---|
| trash-cli | Replaces `rm` usage with `trash-put` (XDG Trash spec) | Deletion only | `trash-restore` | Requires the user to *change habits*; typing `rm` is unprotected |
| rip (rm-improved) | Alternative deletion command, moves files to a "graveyard" (default under `/tmp`) | Deletion only | `rip -u` | Not a drop-in `rm`; graveyard in `/tmp` is lost on reboot |
| safe-rm | Drop-in `rm` wrapper that refuses to delete blocklisted system paths | Deletion of listed paths only | None (prevention, not undo) | No snapshot/undo; only protects a static blocklist |
| trashy | `rm`-like CLI over the freedesktop trash | Deletion only | `trash restore` | Same habit-change problem; Linux-oriented |
| oops-cli | Post-hoc "undo my last terminal command" helper | Varies | Suggests inverse commands | Cannot restore data that was never snapshotted (e.g. `rm` without prior state) |
| macOS Time Machine local snapshots (`tmutil localsnapshot`) | Whole-volume APFS snapshots | Everything, hourly granularity | Mount + copy back | Not per-command; hourly windows lose recent work; requires TM enabled |
| Finder Trash | GUI move-to-Trash | GUI deletions only | GUI restore | Does not cover the shell at all |

## Observations that shape this product

1. **Every existing shell tool alters the command or requires new habits.** trash-cli/rip/trashy ask the user to stop typing `rm`. safe-rm intercepts but changes `rm` semantics (refusal). None protect `mv`/`cp` overwrites, `sed -i`, or `chmod`/`chown`.
2. **Nothing snapshots *before* an arbitrary destructive command runs.** The gap `undo` fills: an *observer* that snapshots affected paths pre-execution and never modifies the command itself (see ADR-001).
3. **Restore-side UX matters.** trash-cli's `trash-restore` (pick from a list) is the pattern users understand; we adopt "most recent by default, pick by id otherwise".
4. **`/tmp`-based stores (rip) are a data-loss anti-pattern.** Our store lives under `$XDG_DATA_HOME` with explicit retention (ADR-006).

## Name collision check

- `undo` on crates.io is an existing unrelated library crate (undo/redo data structures). We do **not** publish to crates.io in v1 (distribution is GitHub Releases + Homebrew tap, per the product decision), so this is informational only.
- No `undo` formula exists in homebrew-core as of this survey; our tap (`<owner>/tap/undo`) is namespaced regardless.
- No widely-adopted shell tool named `undo` was found; `undo doctor` will still check `which -a undo` for local alias/function shadowing.

## Sources

- https://github.com/nivekuil/rip
- https://adamheins.com/blog/a-safer-rm
- https://oops-cli.com/blog/undo-terminal-commands
- https://medium.com/@anuradha99n/%EF%B8%8F-trash-cli-a-safer-alternative-to-rm-for-terminal-users-414ededdaf0a
- https://launchpad.net/safe-rm (via survey article)
