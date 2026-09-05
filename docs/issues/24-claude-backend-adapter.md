# Title

Claude Code backend adapter (`claude -p`)

## Summary

Implement the `claude` GenerationBackend: version-aware invocation using
`claude --bare -p … --permission-mode dontAsk --output-format json` with the payload on
stdin and JSON `.result` extraction.

## Context

Invocation surface verified 2026-07-07 (`docs/research/agent-cli-backends.md`):
`--bare` is the recommended scripted mode but is version-dependent; stdin is capped at
10 MB by the CLI (skillsmith payload budget stays far below).

## Scope

- `src/backend/adapters/claude.ts` only.

## Detailed Requirements

1. `detect()`: `claude --version` (10 s timeout); parse semver; record whether the
   version supports `--bare` (feature-detect: presence in `claude --help` output —
   never assume from version alone; cache per detect call).
2. `buildInvocation({payloadFile, model, timeoutMs})`:
   - Base argv: `claude -p <instruction> --permission-mode dontAsk
     --output-format json` (+ `--model <model>` when configured; + `--bare` prepended
     when supported).
   - `<instruction>` is a short fixed pointer prompt (single sentence, defined in 27:
     "Follow the instructions provided on stdin…") — the full payload goes via
     **stdin** (`stdinFrom: 'payloadFile'`), keeping argv small and stable.
   - `resultFrom: 'stdout-json-result'` (runner extracts `.result`).
3. Failure mapping: stdout parses but `.result` missing/empty → parse-failure with
   `subtype`/`is_error` fields from the CLI JSON quoted when present; stderr
   mentioning authentication → include `claude login`-style hint (mirror 23's
   pattern).
4. Tool-use defense: with `dontAsk` and no allowed tools, a tool attempt aborts the
   run (research doc) — map that CLI outcome to a distinct message telling the user
   the backend attempted tool use and the prompt forbids it (this signals prompt
   drift; reference issue 27).
5. Payload size guard: refuse (usage error) payloads > 9 MiB before spawning (CLI's
   10 MB stdin cap, research doc).
6. **Flag re-verification (mandatory acceptance step)**: same procedure as issue 23
   for `--bare`, `-p`, `--permission-mode dontAsk`, `--output-format json`,
   `--model`; record verified version in header comment; update research doc + DESIGN
   §11.2 on drift.

## Acceptance Criteria

- Shim-based unit tests: `--bare`-supported and unsupported help outputs produce
  correct argv (snapshot); `.result` extraction; `is_error: true` JSON → parse-failure
  with quoted subtype; oversized payload rejected pre-spawn.
- mac-local integration with real CLI: 1-paragraph payload → non-empty `.result`;
  evidence in PR.
- Header comment records verified `claude --version`.

## Validation

`npm test` + `npm run test:local` evidence.

## Dependencies

22.

## Non-goals

Using the Agent SDK packages (CLI only in v1), session continuation (`--continue`),
structured `--json-schema` outputs (sentinel contract instead — 28).

## Design References

- DESIGN §11.2; ADR-003; docs/research/agent-cli-backends.md
