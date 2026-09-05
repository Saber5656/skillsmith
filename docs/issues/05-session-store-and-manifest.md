# Title

Session store: directory layout, manifest state machine, locking

## Summary

Implement `src/store/`: session directory creation with enforced permissions, atomic
manifest reads/writes with the lifecycle state machine, the single-active-session lock,
and list/load/delete APIs.

## Context

DESIGN §5.1 (layout), §5.5 (manifest), §5.6 (state machine) and ADR-005 are normative.
Every command and the daemon go through this module; path-safety rules here back
security boundary B4.

## Scope

- `src/store/sessionStore.ts`, `src/store/manifest.ts`, `src/store/locks.ts`.
- No CLI wiring (07), no prune policy (33 — but `deleteSession` primitive lands here).

## Detailed Requirements

1. `SessionId`: generator `ss-<YYYYMMDD>-<HHmmss>-<4 hex>` (local time) and validator
   regex `^ss-\d{8}-\d{6}-[0-9a-f]{4}$`. **Every** public API taking an id validates it
   before any path join (B4).
2. `createSession({note?, projectDirs}): SessionHandle` — creates
   `sessions/<id>/` `0700` with `ingest/ screens/ ocr/ diffs/` subdirs, writes initial
   manifest (state `created`, DESIGN §5.5 shape exactly), `0600` files.
3. Manifest I/O: `readManifest(id)`, `updateManifest(id, mutator)` — read-modify-write
   with `manifest.json.tmp` + `rename` (atomic), zod-validated both directions.
4. State machine: `assertTransition(from, to)` implementing exactly the DESIGN §5.6
   table (`created→recording`, `recording⇄paused`, `recording|paused→stopping`,
   `stopping→stopped`, `created→failed`, plus recovery `recording|paused→stopped` with
   `stopReason: "crash-recovered"`). Illegal transition → typed `SessionStateError`.
5. Locking (`locks.ts`):
   - `acquireActiveLock(sessionId, pid)`: create `active-session.lock` with `O_EXCL`;
     on success write `active-session.json` `{sessionId, pid, startedAt}`.
   - `releaseActiveLock(sessionId)`: verifies owner id matches before unlink.
   - `readActiveSession(): {sessionId, pid} | null` (also used by the zsh hook — the
     JSON must stay flat and stable).
   - Stale detection helper: lock exists but `kill(pid, 0)` fails → `{stale: true}`.
6. `listSessions(): SessionSummary[]` (id, state, createdAt, sizeBytes lazily computed,
   counters from manifest) sorted by id descending.
7. `deleteSession(id, {force})`: refuses `recording|paused|stopping` unless `force`;
   resolves the real path, verifies it is strictly inside `sessionsDir()` and not a
   symlink (`lstat`), then removes recursively.
8. Permission enforcement: every dir/file creation sets modes explicitly; a
   `auditPermissions(id)` helper returns violations (consumed by doctor, issue 32).

## Acceptance Criteria

- Unit tests on a temp `XDG_DATA_HOME`: create → transitions happy path; every illegal
  transition rejected; atomicity (simulated crash: `.tmp` present → last valid manifest
  wins); lock contention (second acquire fails); stale lock detection; delete safety
  (symlinked session dir refused; id `../../etc` rejected by regex).
- `listSessions` on 50 generated sessions returns correct order and states.
- All created paths verified `0700`/`0600` in tests (`fs.stat` mode checks).

## Validation

`npm test`; manual: create two sessions in a scratch HOME, corrupt one manifest, confirm
readable error naming the file.

## Dependencies

01, 03, 04.

## Non-goals

Daemon/heartbeat writing (07), prune policy (33), redacted/generated artifact management
(20, 30).

## Design References

- DESIGN §5.1, §5.2, §5.5, §5.6, §14.1 B4; ADR-005
