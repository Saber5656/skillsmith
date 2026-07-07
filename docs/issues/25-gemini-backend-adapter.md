# Title

Gemini CLI backend adapter (`gemini -p`)

## Summary

Implement the `gemini` GenerationBackend with an explicit implementation-time
verification of stdin piping, and an argv-embedding fallback guarded by size limits.

## Context

Research (2026-07-07) confirmed `gemini -p`, `--output-format json`, `-m`, but did
**not** confirm stdin piping — this is a tracked known unknown (DESIGN §16 item 3)
that this issue must resolve and document.

## Scope

- `src/backend/adapters/gemini.ts` only.

## Detailed Requirements

1. `detect()`: `gemini --version` (10 s timeout).
2. **Stdin verification step (mandatory, first task)**: with the installed CLI, test
   `printf 'say ok' | gemini -p "Follow stdin instructions" --output-format json`
   behavior. Record the outcome in `docs/research/agent-cli-backends.md` (update the
   Gemini table) and choose mode:
   - stdin works → `stdinFrom: 'payloadFile'`, pointer prompt in `-p` (same pattern as
     24).
   - stdin unsupported → fallback: embed the payload as the `-p` argument; enforce a
     conservative 200 KiB argv guard (macOS ARG_MAX headroom); larger payloads →
     `BackendError` instructing the user to lower `generate.maxPayloadChars` or choose
     another backend.
3. argv: `gemini -p <prompt> --output-format json` (+ `-m <model>` when configured);
   `resultFrom: 'stdout-json-result'` — verify the JSON shape's text field name during
   the same verification step (research doc notes `response`-style fields may differ;
   adapter maps whichever field the installed version emits, preferring a documented
   one; record it).
4. Sandbox/approval flags: investigate `--sandbox` / approval flags during
   verification; if a flag prevents tool execution in non-interactive mode, add it and
   document; otherwise rely on the prompt + parse strictness (same posture as 24 req 4).
5. Error mapping mirrors 23/24 including auth hint.

## Acceptance Criteria

- Verification outcomes written into `docs/research/agent-cli-backends.md` (mode
  chosen, JSON field, flags, CLI version) in the same PR.
- Shim-based unit tests for the chosen mode + the argv-size guard (fallback mode is
  tested regardless, behind a mode switch).
- mac-local integration with real CLI: 1-paragraph payload → non-empty result;
  evidence in PR.
- Header comment records verified `gemini --version`.

## Validation

`npm test` + `npm run test:local` evidence.

## Dependencies

22 (pattern-parity review against 23/24 recommended).

## Non-goals

Google Cloud auth management, Vertex endpoints, multimodal payloads (v2 candidate for
sending screenshots directly).

## Design References

- DESIGN §11.2, §16; ADR-003; docs/research/agent-cli-backends.md
