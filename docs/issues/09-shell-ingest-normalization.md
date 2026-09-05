# Title

Shell ingest reader and event normalization (daemon side)

## Summary

Implement the daemon-side recorder that tails `ingest/shell-*.jsonl` (and
`ingest/markers.jsonl`), pairs pre/post records, and emits normalized `shell.command`
and `session.marker` events into `events.jsonl`.

## Context

The zsh hook (08) writes raw, per-shell append files (multi-writer safety per ADR-005).
This recorder converts them into the single normalized log. DESIGN §7.1 defines pairing
rules; §5.4 the target event shape.

## Scope

- `src/record/shell/ingestReader.ts` registered as a `Recorder` (07 registry).

## Detailed Requirements

1. Poll the `ingest/` directory every 1 s for `shell-*.jsonl` files; per file keep a
   byte offset; read only appended bytes (partial last line buffered until newline).
2. Parse each line as the raw hook schema (`{e:"pre"|"post", id, ts, cwd?, cmd?, code?,
   truncated?}`); malformed lines → increment a per-session `ingestCorrupt` counter
   (exposed via heartbeat counters), continue.
3. Pairing: `pre` opens a pending entry keyed `(file, id)`; matching `post` closes it →
   emit `shell.command` with `startedAt` (pre.ts), `endedAt` (post.ts),
   `durationMs` (delta, clamp ≥ 0), `exitCode`, `cwd`, `command`, `tty` (from filename),
   `shell: "zsh"`, `truncated`.
4. Unpaired `pre` older than 10 min → emit with `endedAt: undefined, exitCode: null`
   (long-running command or shell killed); unpaired `post` → drop + corrupt counter.
5. `ts` from the hook is epoch seconds with fraction (`EPOCHREALTIME`); convert to ISO
   UTC ms. Reject events with ts outside the session's `[startedAt − 60 s, now + 60 s]`
   window (clock sanity; drop + corrupt counter).
6. Marker ingestion: `ingest/markers.jsonl` lines `{label, ts}` → `session.marker`
   events (same tail mechanism).
7. On recorder `stop()`: one final drain pass, then flush all still-pending `pre`
   entries as unpaired (rule 4 without the age requirement).
8. Backpressure: cap pending map at 1000 entries (oldest flushed unpaired);
   cap per-poll processed lines at 5000 (rest next tick).

## Acceptance Criteria

- Fixture-driven tests: interleaved multi-file ingest (two shells) normalizes to
  correctly ordered, correctly paired events; unpaired/malformed cases per rules 3–5;
  final-drain on stop.
- Live test with the real hook (mac-only): run 5 commands in a scripted zsh while the
  daemon records → `events.jsonl` contains 5 `shell.command` events with sane
  durations, plus start/stop bookkeeping.
- Counter surfacing: corrupt ingest lines appear in `record status --json` output.

## Validation

`npm test`; the mac-only live test runs in `test:local` and on the macOS CI job
(zsh present on macos-14 runners).

## Dependencies

04, 05, 07, 08.

## Non-goals

Capturing command output/stdout, bash support, semantic command classification (the
digest layer handles presentation — 16/18).

## Design References

- DESIGN §5.4, §7.1; ADR-005
