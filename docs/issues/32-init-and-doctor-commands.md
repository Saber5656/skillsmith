# Title

`skillsmith init` (guided setup) and `skillsmith doctor` (diagnostics)

## Summary

Implement the interactive first-run setup and the environment diagnosis command,
including the Screen Recording permission walkthrough, shell hook management, backend
detection, and store health checks.

## Context

DESIGN §13.1 and §13.3 are normative; the doctor check table is the contract. init
composes helpers from 03 (config), 08 (hook installer), 22 (backend detect), and the
capture probe from 10.

## Scope

- `src/cli/commands/init.ts`, `doctor.ts`; a shared `src/doctor/checks.ts` registry.

## Detailed Requirements

1. `init` (interactive; every mutation individually confirmed):
   1. Config: if absent, show the default config (DESIGN §4) and confirm writing the
      scaffold (commented YAML) to the config path (0600); existing config → skip with
      note.
   2. Shell hook: show the exact `~/.zshrc` block, confirm install (08); already
      current → note.
   3. Screen permission: run a probe capture to a temp file (10's detection triple);
      on failure print the guidance and offer to run
      `open "x-apple.systempreferences:com.apple.preference.security?Privacy_ScreenCapture"`,
      then re-probe on Enter (loop ≤ 3 attempts, skippable).
   4. OCR: report resolved engine per ADR-006 (`auto` resolution result).
   5. Backends: `detectAll()` (22) table; if none available print install hints for
      all three CLIs; if `backend.default` unavailable, offer switching the config
      default to a detected one (confirmed write).
   6. Summary + next steps (`record start`, docs link).
   - `--no-input`: performs config scaffold (if absent) + read-only checks only; no
     zshrc/system mutations; prints what interactive mode would do. Exit 0 even with
     missing permissions (report-only).
   - `--uninstall-hook` subflag: runs 08's uninstaller (confirmed).
2. `doctor [--json] [--fix]`: run the check registry, table output
   `check | status(ok/warn/fail) | detail | fix hint`; checks exactly per DESIGN §13.3:
   config parse; data-dir permissions (with `--fix` → chmod 0700/0600 after listing);
   hook status (08); screen capture probe; OCR engine resolution; backend detection;
   active-session/lock consistency (stale lock → offer recovery via 07's routine, only
   with `--fix`); store disk usage vs `retentionDays`/size (warn + prune hint);
   `backend.custom` configured → informational warning (26's notice).
   - Exit 0 all ok/warn; exit 4 when any check fails (`--json`: full structured
     results regardless).
3. Both commands must work before any session exists (empty store) and on a fully
   populated store.
4. All prompts via a single readline helper (stdin TTY detection; non-TTY + not
   `--no-input` → exit 2 telling the user to use `--no-input`).

## Acceptance Criteria

- doctor tests with dependency injection (fake checks environment): each check's
  ok/warn/fail path + exit codes + `--json` schema snapshot; `--fix` applies chmod and
  stale-lock recovery and is idempotent.
- init `--no-input` on a clean HOME creates only the config file (fs diff assertion)
  and exits 0.
- Interactive init happy path driven by scripted stdin: hook installed, config
  written, summary printed (zshrc content asserted via 08's tests helpers).
- Permission-probe failure path prints the deep link command without executing it in
  tests (subprocess stub).
- mac-local: real `doctor` output screenshot in PR from a configured machine.

## Validation

`npm test` + manual `init` walkthrough evidence (fresh user account or scratch HOME).

## Dependencies

03, 05, 06, 07, 08, 10 (probe), 11 (engine resolution), 22 (detect), 26 (notice).

## Non-goals

Automated TCC granting (impossible by design), config editing UI, migration of old
stores.

## Design References

- DESIGN §13.1, §13.3; ADR-006; docs/research/macos-capture-and-ocr.md
