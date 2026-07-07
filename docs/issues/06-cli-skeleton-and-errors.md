# Title

CLI skeleton, error taxonomy, exit codes, logging

## Summary

Implement the command framework in `src/cli/` (commander), the typed error hierarchy
mapped to the stable exit-code table, global flags (`--json`, `--verbose`, `--quiet`),
and the secret-safe logger.

## Context

DESIGN §3 defines the command surface and §3.1 the exit-code contract; §15.1 requires
logs to pass through high-severity redaction so captured secrets never leak via logs.
All later command issues plug into this skeleton.

## Scope

- `src/cli/index.ts`, `src/cli/context.ts`, `src/util/errors.ts`, `src/util/logger.ts`.
- Register all v1 commands as stubs that print `not implemented` and exit 1 — the
  full command list from DESIGN §3 with their documented flags, so `--help` is complete
  from day one.

## Detailed Requirements

1. Commander program: name/description/version; subcommands exactly per DESIGN §3
   including nested `record start|stop|status|pause|resume`, `sessions list|show|delete|prune`,
   `config show|path`.
2. Global flags on every command: `--json` (machine output → stdout; all human/log text
   → stderr), `--verbose` (debug logs), `--quiet` (errors only). `--json` and non-JSON
   human output are mutually exclusive code paths — commands receive an `Output` helper
   (`out.human(str)`, `out.json(obj)`) from `context.ts`.
3. Error taxonomy in `errors.ts`: `UsageError(2)`, `SessionStateError(3)`,
   `MissingDependencyError(4)`, `ConsentDeclinedError(5)`, `BackendError(6)`,
   `ValidationFailedError(7)`, base `SkillsmithError(1)`. Single top-level handler maps
   error → exit code, prints `error: <message>` (+ stack with `--verbose`), and for
   `--json` mode emits `{"error": {"code": <int>, "name": "...", "message": "..."}}`.
4. Logger (`logger.ts`): leveled (`debug|info|warn|error`) to stderr with ISO
   timestamps in verbose mode. Constructor takes a `sanitize(text) => text` hook; wire a
   placeholder identity now and replace with the high-severity redaction subset in
   issue 20 (leave a typed TODO referencing issue 20).
5. Exit-code integration test harness: helper `runCli(argv): {code, stdout, stderr}`
   using child_process on the built CLI — this helper is reused by all command tests.
6. `skillsmith --version` prints the package version; unknown command → usage error
   exit 2 with help hint.

## Acceptance Criteria

- `skillsmith --help` lists every DESIGN §3 command with flags; snapshot test.
- Each error class exits with its documented code (table-driven test via `runCli`).
- `--json` on a stub command emits the structured error object, nothing on stdout
  besides JSON.
- Logger writes only to stderr; verified in tests.

## Validation

`npm test`; manual `--help` review pasted into PR.

## Dependencies

01. (03/05 consume this; ordering with 03 is flexible — see ISSUE_PLAN.)

## Non-goals

Real command behavior (07+), shell completion (v2), i18n (English-only v1).

## Design References

- DESIGN §3, §3.1, §15.1
