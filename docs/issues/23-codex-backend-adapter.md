# Title

Codex backend adapter (`codex exec`) — default backend

## Summary

Implement the `codex` GenerationBackend: detection, invocation via
`codex exec --ephemeral … -o <file> -` with the payload on stdin, and error mapping.

## Context

Default backend per DESIGN §4 (`backend.default: codex`). Invocation surface verified
2026-07-07 in `docs/research/agent-cli-backends.md`; CLI drift is a tracked risk with a
mandatory re-verification step.

## Scope

- `src/backend/adapters/codex.ts` only (runner from 22 does execution).

## Detailed Requirements

1. `detect()`: `codex --version` via runner-style execFile (10 s timeout); parse
   version string; `available: true` + `path` (resolved from PATH) + `version`.
2. `buildInvocation({payloadFile, model, timeoutMs})`:
   - argv: `codex exec --ephemeral --skip-git-repo-check -o <cwd>/last-message.md -`
     with `-m <model>` inserted **only** when `config.backend.codex.model` (or request
     model) is set.
   - `stdinFrom: 'payloadFile'`, `resultFrom: 'file'`,
     `resultFile: '<cwd>/last-message.md'` (inside the runner's temp cwd — the runner
     substitutes its actual cwd path; define the contract as a `${CWD}` placeholder
     resolved by the runner).
   - Rationale comments in code: `--ephemeral` (no rollout persistence — privacy,
     ADR-004 adjacent), `--skip-git-repo-check` (empty temp cwd is not a repo),
     read-only default sandbox (research doc).
3. Empty-result handling: `last-message.md` missing or empty → `parse-failure`
   BackendError including the stdout/stderr tail (sanitized) to help diagnose auth
   issues (`codex login` hint in the message when stderr mentions auth).
4. **Flag re-verification (mandatory acceptance step)**: run `codex exec --help`
   against the installed CLI at implementation time; confirm `--ephemeral`,
   `--skip-git-repo-check`, `-o/--output-last-message`, `-` stdin, `-m`; record the
   verified CLI version in a header comment; on drift: update
   `docs/research/agent-cli-backends.md` + DESIGN §11.2 table in the same PR.

## Acceptance Criteria

- Unit tests with a fixture `codex` shim (node script mimicking: reads stdin, writes
  `last-message.md`, exits 0): happy path returns file content; missing file →
  parse-failure; non-zero exit → non-zero-exit taxonomy; model flag included/omitted
  correctly (argv snapshot tests).
- Auth-failure shim (stderr `Not logged in`, exit 1) → error message contains the
  login hint.
- mac-local integration (`test:local`, requires real logged-in codex): a 1-paragraph
  payload returns non-empty text; recorded as evidence in PR (output excerpt).
- Header comment records verified `codex --version`.

## Validation

`npm test` + `npm run test:local` evidence in PR.

## Dependencies

22.

## Non-goals

Prompt content (27), output parsing into files (28), retries, codex config management
(`--ignore-user-config` decision documented in research doc: not used).

## Design References

- DESIGN §11.2; ADR-003; docs/research/agent-cli-backends.md
