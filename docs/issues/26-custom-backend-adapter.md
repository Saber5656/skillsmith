# Title

Custom command backend adapter (config-defined argv)

## Summary

Implement the `custom` GenerationBackend: run a user-configured command
(`backend.custom.argv`) as the generation backend, supporting stdin or argument prompt
delivery. This also provides the deterministic fake backend for E2E tests (35).

## Context

ADR-003 names `custom` as the escape hatch for other CLIs (e.g. `ollama run`) and the
test fixture mechanism. Boundary B9 (DESIGN §14.1): config is trusted local input, but
setting a custom backend means "skillsmith runs this command as you" — that must be
visible, not silent.

## Scope

- `src/backend/adapters/custom.ts`; doctor warning wiring lands with 32 via an
  exported `customBackendNotice()` helper.

## Detailed Requirements

1. Config (schema already in 03): `backend.custom.argv: string[]` (non-empty when the
   backend is selected; else selecting `custom` → exit 4 with config hint),
   `backend.custom.promptVia: 'stdin' | 'arg'`.
2. `detect()`: `available` iff `argv[0]` resolves on PATH or is an existing executable
   absolute path; `version: undefined`.
3. `buildInvocation`:
   - `promptVia: 'stdin'` → argv as configured verbatim, `stdinFrom: 'payloadFile'`.
   - `promptVia: 'arg'` → argv + payload appended as one final argument (runner reads
     the payload file and appends; 200 KiB argv guard as in 25).
   - `resultFrom: 'stdout-raw'`.
   - No shell interpretation ever — argv passes to the runner untouched; document
     that pipes/redirection in argv strings will not work (write a wrapper script
     instead).
4. Model field: `backend.custom` has no model passthrough in v1 (users bake it into
   argv) — documented.
5. First selection of `custom` in a `generate` run prints a one-line notice (stderr):
   `using custom backend command: <argv[0]> (configured in <config path>)`.

## Acceptance Criteria

- Unit tests: stdin mode with a fixture echo-backend; arg mode with size guard; empty
  argv → exit-4 typed error; PATH vs absolute resolution; notice printed once.
- The fixture fake backend for E2E (deterministic sentinel output; lands fully in 35)
  is exercised end-to-end here at unit scale: payload in → canned sentinel out.
- Snapshot test proving argv is not shell-expanded (argument containing `$(rm -rf /)`
  arrives literally at the child — fixture dumps argv).

## Validation

`npm test`.

## Dependencies

22 (03 schema field must exist — it does per issue 03).

## Non-goals

Templating/placeholders in argv (v2 if requested), per-custom-backend result parsing
formats (raw stdout only).

## Design References

- DESIGN §11.2, §14.1 B9; ADR-003
