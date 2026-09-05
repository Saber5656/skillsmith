# Title

Record lifecycle commands and recording daemon process management

## Summary

Implement `record start|stop|status|pause|resume`, `mark`, `sessions list|show|delete`,
and the detached daemon skeleton with heartbeat, signal handling, control-file
pause/resume, limits enforcement, and crash recovery.

## Context

DESIGN §5.6 (state machine) and §6 (daemon) are normative. Recorders (issues 08–15)
plug into the daemon's recorder registry; this issue delivers the lifecycle with a
no-op recorder set so it is independently testable.

## Scope

- `src/record/daemon/daemon.ts`, `lifecycle.ts`, `heartbeat.ts`,
  `src/cli/commands/record.ts`, `sessions.ts`, `mark.ts`.
- Recorder registry interface: `Recorder {id, start(ctx), stop(): Promise<void>}` with
  a `RecorderContext {emitEvent(kind, data), sessionDir, config, state$}` — concrete
  recorders come in later issues.

## Detailed Requirements

1. `record start [--project <dir>]... [--note <text>]`:
   - Preconditions: no active lock (else exit 3 with the active session id); project
     dirs exist and are directories (resolve to absolute; default `[cwd]`).
   - Create session (05), acquire lock, spawn `process.execPath dist/cli/index.js
     __daemon <id>` detached with stdio to `daemon.log`, `unref()`.
   - Wait ≤ 10 s for manifest `state: recording` (poll 200 ms); on timeout: mark
     `failed`, release lock, exit 4 with `daemon.log` tail (last 20 lines).
2. Hidden `__daemon <id>` command:
   - Assigns the global `seq` counter; is the **only** writer of `events.jsonl`.
   - Emits `session.start`, transitions manifest → `recording`, starts recorders
     (registry may be empty), heartbeat every 5 s (atomic manifest update: timestamps +
     counters + recorder states).
   - Polls `control.json` (1 s): `{desired: "paused"}` → stop capture loops, emit
     `session.pause`, state `paused`; reverse for resume.
   - Limits: session duration > `maxSessionHours` → graceful self-stop with
     `session.stop{reason:"limit-hours"}`; store size checked each minute — over
     `maxSessionSizeMB` → pause capture + `session.warning{code:"size-limit"}` once.
   - SIGTERM/SIGINT: stop recorders → drain queues ≤ 30 s → `session.stop{reason:user}`
     → state `stopped` → release lock → exit 0. Second signal → flush minimal + exit 1.
3. `record stop`: read active session; no active → exit 3. SIGTERM to pid; poll ≤ 15 s
   for `stopped`; else SIGKILL, run crash-recovery finalization (below), report.
4. Crash recovery (shared routine, also run by `status`/`doctor`): if manifest state is
   `recording|paused` and heartbeat older than 60 s and pid dead → finalize: append
   `session.stop{reason:"crash-recovered"}`, state `stopped`, release lock.
5. `record status [--json]`: state, uptime, counters, recorder states, warnings; exit 3
   if none active (human message "no active session").
6. `record pause|resume`: write `control.json` atomically; wait ≤ 5 s for state change.
7. `mark <label>`: append a marker request via `ingest/markers.jsonl` (daemon ingests
   → `session.marker` event); label ≤ 512 chars; requires active recording session.
8. `sessions list|show|delete` wiring to store APIs (05): `show` prints manifest +
   counters + last 5 warnings; `--json` full manifest.

## Acceptance Criteria

- E2E-ish test (temp HOME, built CLI): start (empty recorder set) → status shows
  `recording` → pause → resume → mark → stop → manifest `stopped`, events contain
  start/pause/resume/marker/stop in order, lock released.
- Kill -9 the daemon; `record status` finalizes to `crash-recovered` and releases lock.
- Second `record start` while active exits 3.
- `record start` with missing project dir exits 2 (usage) naming the dir.
- Daemon survives `screencapture`-less environments (no recorders registered) — no
  spurious errors in `daemon.log`.

## Validation

`npm test` including the lifecycle test above; manual smoke on a real machine with
`--verbose`, attach `daemon.log`.

## Dependencies

03, 04, 05, 06.

## Non-goals

Actual capture (08–15), OCR draining specifics (11), prune (33).

## Design References

- DESIGN §3, §5.5, §5.6, §6
