# Title

Generation backend abstraction, registry, and hardened subprocess runner

## Summary

Implement `src/backend/types.ts`, `registry.ts`, and `runner.ts`: the
`GenerationBackend` interface, backend selection/detection, and the shared hardened
subprocess execution layer used by all adapters.

## Context

ADR-003 (delegation model) and DESIGN §11.1 are normative; boundary B7 (DESIGN §14.1)
defines the isolation requirements. Adapters (23–26) implement only
`detect`/`buildInvocation` against this layer.

## Scope

- Interface + registry + runner + error taxonomy. No concrete adapters.

## Detailed Requirements

1. Types exactly per DESIGN §11.1 (`GenerationBackend`, invocation descriptor with
   `argv`, `stdinFrom: 'payloadFile' | null`, `resultFrom: 'stdout-json-result' |
   'file' | 'stdout-raw'`, `resultFile?`).
2. Registry: `getBackend(id)`, `listBackends()` (static registration of 23–26;
   unknown id → `MissingDependencyError` exit 4 naming valid ids). Selection order in
   commands: `--backend` flag > `config.backend.default`.
3. Runner `runBackend(invocation, {payloadFile, timeoutMs}): {stdout, stderr, exitCode,
   resultText}`:
   - `execFile` semantics — argv array, `shell: false` always.
   - `cwd`: fresh `fs.mkdtemp` under `os.tmpdir()`, `chmod 0700`, recursively removed
     in `finally`.
   - `env`: exactly `{PATH, HOME, LANG, LC_ALL, TERM: 'dumb'}` from the parent —
     nothing else; assert no `SKILLSMITH_*` leaks.
   - stdin: stream the payload file when `stdinFrom: 'payloadFile'`, else close stdin.
   - Timeout → SIGTERM to the child's process group (spawn with `detached: true` to own
     the group), 5 s grace, SIGKILL; raise `BackendError('timeout')`.
   - stdout/stderr capped at 10 MiB each (rolling; over-cap → kill +
     `BackendError('output-too-large')`).
   - Result extraction: `file` → read `resultFile` (must exist, ≤ 10 MiB);
     `stdout-json-result` → parse stdout as JSON, take `.result` string;
     `stdout-raw` → stdout text. Missing/invalid → `BackendError('parse-failure')`
     with stderr excerpt (sanitized via logger rules) in the message.
   - Error taxonomy: `not-found` (ENOENT), `timeout`, `non-zero-exit` (include code +
     sanitized stderr tail ≤ 2 KiB), `output-too-large`, `parse-failure` — all mapping
     to exit 6 via `BackendError`.
4. `detectAll(): [{id, available, version?, path?}]` for doctor (32) — parallel with
   per-detect 10 s timeout.
5. Never write anything into the temp cwd besides what the child creates; skillsmith
   passes the payload only via stdin or `resultFile` paths **outside** the temp dir are
   forbidden (resultFile must live inside the temp cwd — enforce by path check).

## Acceptance Criteria

- Runner tests using fixture executables (small node scripts in `test/fixtures/bin/`):
  happy path (all three resultFrom modes); timeout kills a sleeping child **and** its
  spawned grandchild (process-group verified); output cap triggers; ENOENT taxonomy;
  non-zero exit taxonomy with stderr tail; env whitelist (child dumps env, assert
  exactly the five keys); temp cwd created 0700 and removed even on failure.
- Registry: unknown backend id → exit-4 typed error listing valid ids.
- resultFile path escape attempt (fixture sets `../x`) → rejected.

## Validation

`npm test` (fixture-executable approach is platform-neutral).

## Dependencies

03, 06.

## Non-goals

Concrete adapters (23–26), retries (v1: single attempt; rerun is user-level), cost
accounting (v2).

## Design References

- DESIGN §11.1, §14.1 B7; ADR-003
