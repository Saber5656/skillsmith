# Title

Zsh shell hook and hook installer/uninstaller

## Summary

Ship the zsh hook script that records shell commands into per-shell ingest files, and
the installer that manages a marked block in `~/.zshrc`.

## Context

DESIGN §7.1 is normative; security boundary B1 (DESIGN §14.1) demands the hook be
fail-silent, append-only, and instantly disableable. The hook writes raw ingest files;
normalization is issue 09.

## Scope

- `assets/skillsmith-hook.zsh` (shipped in the npm package), `src/record/shell/installer.ts`,
  wiring into `init` (32) and `doctor` checks (32) via exported helpers.

## Detailed Requirements

1. Hook script (`assets/skillsmith-hook.zsh`), pure zsh:
   - Loads `zsh/datetime`; defines `__skillsmith_preexec` and `__skillsmith_precmd`,
     registered via `add-zsh-hook` (autoload) — never clobbers existing hook arrays.
   - Fast-path exit conditions checked in order: `$SKILLSMITH_DISABLE == 1`; pointer
     file `~/.local/share/skillsmith/active-session.json` missing (honor
     `XDG_DATA_HOME`); pointer unreadable.
   - Pointer parse without spawning processes: extract `sessionId` with zsh pattern
     matching from the flat JSON (the store guarantees flat stable JSON — DESIGN
     issue 05 req 5). Cache the parsed id + pointer mtime in shell variables; re-parse
     only when mtime changes (one `zstat` per prompt, from `zsh/stat`).
   - `preexec`: generate `cmdId="$$-$EPOCHREALTIME-$RANDOM"`; JSON-escape the command
     (backslash, quote, control chars) in pure zsh; truncate to
     `maxCommandBytes` (hardcode 8192; config not readable from the hook) appending
     `"…"` and `"truncated":true`; append one line
     `{"e":"pre","id":…,"ts":…,"cwd":…,"cmd":…}` to
     `<data>/sessions/<id>/ingest/shell-${TTY##*/}-$$.jsonl`.
   - `precmd`: if a `pre` was written this cycle, append
     `{"e":"post","id":…,"ts":…,"code":$?}`.
   - **Every** filesystem operation suffixed `2>/dev/null || return 0`. The hook must
     never print, never block on locks, never exit the shell.
   - First line comment: `# skillsmith hook v1 — managed block, do not edit`.
2. Installer (`installer.ts`):
   - `installHook()`: append to `~/.zshrc` (create if absent) the block
     `# >>> skillsmith >>>` / `source <abs path to installed assets/skillsmith-hook.zsh>`
     / `# <<< skillsmith <<<`; idempotent (existing block with same content → no-op;
     different content → replace in place).
   - `uninstallHook()`: remove the marked block only; refuse if markers are unbalanced.
   - `hookStatus()`: `installed-current | installed-outdated | not-installed` (hash of
     block content vs expected).
   - Never modifies anything outside the marker block; preserves file mode; writes via
     tmp+rename.
3. The asset path must resolve correctly from the globally installed npm package
   (compute from `import.meta.url`, not cwd).

## Acceptance Criteria

- Scripted zsh test (spawn `zsh -i` with a temp `ZDOTDIR`/HOME): with active pointer,
  running `echo hi` produces a valid `pre`+`post` pair with correct cwd, escaped JSON,
  exit code 0; `false` yields `code:1`.
- With no pointer / `SKILLSMITH_DISABLE=1`, zero files are created and the shell shows
  no output difference (assert empty ingest dir and clean stderr).
- Command containing `"`, newline, emoji, and 10 KiB length produces valid JSON with
  `truncated: true`.
- Installer round-trip: install → status current → modify asset hash → status outdated
  → reinstall replaces block → uninstall leaves `~/.zshrc` byte-identical to original.
- Hook adds < 5 ms per prompt with pointer absent (measure with `EPOCHREALTIME` around
  1000 iterations in the test; generous threshold 15 ms on CI).

## Validation

Test suite above is macOS/zsh (name tests `*.mac.test.ts` where zsh-dependent — Ubuntu
runners have zsh only if installed; guard with a zsh presence check). Manual: install in
a real shell, record a short session, inspect ingest files.

## Dependencies

05 (pointer file shape), 07 (session dirs exist during recording).

## Non-goals

bash/fish hooks (v2), ingest normalization (09), capturing command output (v2 PTY).

## Design References

- DESIGN §7.1, §14.1 B1; ADR-001
