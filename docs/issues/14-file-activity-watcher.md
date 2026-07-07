# Title

File activity watcher with bounded diffs

## Summary

Implement the files recorder: a chokidar watcher over the session's project
directories emitting `file.change` events with content hashes and bounded unified
diffs for text files.

## Context

DESIGN §7.4 is normative; boundary B3 (DESIGN §14.1) treats watched content as
untrusted data (size caps, symlink skip, no content execution).

## Scope

- `src/record/files/watcher.ts`, `differ.ts` (Recorder registration per 07).
  Git snapshots are issue 15.

## Detailed Requirements

1. Watch each `projectDirs` entry with chokidar: `ignoreInitial: true`,
   `awaitWriteFinish: {stabilityThreshold: 500, pollInterval: 100}` (the 500 ms
   debounce), depth unlimited, `followSymlinks: false`.
2. Ignore set (merged, in order): built-in defaults
   `['**/node_modules/**','**/.git/**','**/dist/**','**/build/**','**/coverage/**',
   '**/.venv/**','**/__pycache__/**','**/*.min.*','**/.DS_Store','**/*.lock',
   '**/package-lock.json']` ∪ config `recording.files.ignore` ∪ patterns parsed
   best-effort from each project dir's root `.gitignore` (comment/negation lines
   skipped; negations unsupported in v1 — document).
3. Per event (add/change/unlink):
   - `lstat` first; skip symlinks and non-regular files (B3).
   - size > `maxFileSizeBytes` → metadata-only event (`sizeBytes`, no hash/diff).
   - Binary sniff: any NUL byte in first 8 KiB → `isBinary: true`, hash only.
   - Text ≤ cap: sha256 hash; identical hash to last capture of the path → drop
     (dedup). Diff vs previous captured content (in-memory LRU cache, 512 entries,
     value = content; on LRU miss → no diff, hash only) using the `diff` npm package
     unified format; write to `diffs/<seq>.patch` truncated at `maxDiffBytes` with a
     trailing `…truncated` marker and `truncated: true` on the event.
   - `unlink` → `changeType:"delete"`, no content read.
4. Emit `file.change` with a path **relative to its project dir** prefixed by an index
   marker when multiple project dirs exist (`p0:src/a.ts`), absolute paths never stored
   (privacy: home-dir leakage minimized before redaction even runs).
5. Storm control: sliding 1 s window; > 50 events → coalesce per path (keep last),
   emit `session.warning{code:"file-storm"}` once per session; hard cap 10 000
   `file.change` events per session, then metadata-only mode + warning.
6. Watcher errors (EPERM, ENOSPC/inotify-equivalent) → recorder state `error`,
   `session.warning`, other recorders unaffected.

## Acceptance Criteria

- Temp-dir tests: create/modify/delete cycle produces expected events with correct
  relative paths, hashes, and a correct unified diff; rewrite-with-same-content emits
  nothing; binary file (contains NUL) → hash-only; 2 MiB file → metadata-only;
  symlink → no event.
- Ignore tests: files under `node_modules/`, config-ignored glob, and `.gitignore`d
  path produce no events.
- Storm test: 500 rapid writes to 20 files → coalesced events, single warning, daemon
  responsive (heartbeat continues — assert timestamps).
- Diff truncation at `maxDiffBytes` verified byte-exact with marker.

## Validation

`npm test` (all above are platform-neutral; runs on both CI OSes).

## Dependencies

03, 04, 07.

## Non-goals

Git snapshots (15), rename detection (v1 emits delete+create), editor-plugin events
(v2), negated gitignore patterns (v1 limitation, documented).

## Design References

- DESIGN §7.4, §14.1 B3; ADR-001
