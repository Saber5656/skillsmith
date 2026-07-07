# Title

Event model (zod schemas) and JSONL utilities

## Summary

Implement the versioned event envelope, zod schemas for every v1 event kind, and
crash-tolerant JSONL append/read utilities in `src/store/events.ts` and
`src/util/jsonl.ts`.

## Context

`events.jsonl` is the spine of a session (ADR-005). DESIGN §5.3–§5.4 defines the
envelope and the normative field tables. Readers must tolerate crash-truncated tails.

## Scope

- `src/store/events.ts` (schemas + types), `src/util/jsonl.ts` (I/O). No session
  directory management (05), no recorders.

## Detailed Requirements

1. Envelope schema: `{v: literal(1), ts: ISO-8601 UTC string with ms, seq: int ≥ 0,
   source: enum, kind: enum, data: object}` — discriminated union on `kind` matching
   **every row** of DESIGN §5.4 with exact field names, optionality, and enums
   (`session.start`, `session.stop`, `session.pause`, `session.resume`,
   `session.marker`, `session.warning`, `shell.command`, `screen.capture`,
   `screen.skipped`, `screen.ocr`, `file.change`, `git.snapshot`).
2. Exported helpers: `parseEventLine(line): Event | {error}` and
   `serializeEvent(event): string` (single line, no pretty print, key order stable:
   `v, ts, seq, source, kind, data`).
3. `jsonl.ts`:
   - `appendLine(file, line)`: open `a`, single `write()` of `line + "\n"`, mode 0600 on
     create. Expose `appendLineSync` too (daemon flush paths).
   - `readJsonl(file, onLine, onCorrupt)`: streaming line reader (no full-file buffer);
     lines failing JSON.parse or schema validation invoke `onCorrupt(lineNo, reason)`
     and continue. Final partial line without `\n` is treated as corrupt-tail (not an
     error).
   - `readEvents(file): {events: Event[], corruptLines: number}` convenience.
4. `seq` monotonicity is *asserted by readers* (non-monotonic → corrupt count, keep
   going), *guaranteed by the daemon* (issue 07 holds the counter).
5. Size guards: individual line > 1 MiB → corrupt (protects downstream buffers).

## Acceptance Criteria

- Round-trip test for every event kind (construct → serialize → parse → deep-equal).
- Truncated-tail fixture (file ending mid-JSON) parses remaining events and reports
  `corruptLines: 1`.
- Interleaved garbage lines are skipped and counted; ordering preserved.
- 1 MiB+ line fixture counted corrupt without memory blow-up (streaming verified).
- Type exports compile under `strict` with no `any` in public signatures.

## Validation

`npm test`; include a generated 100k-line file performance smoke (< 2 s read on CI
macOS runner, assertion with generous margin).

## Dependencies

01.

## Non-goals

Manifest handling (05), event *production* (07–15), digesting (16–18).

## Design References

- DESIGN §5.3, §5.4; ADR-005
