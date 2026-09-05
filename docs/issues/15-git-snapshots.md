# Title

Git snapshots at session start and stop

## Summary

Capture repository state (branch, HEAD, dirty files, diff stat) for every git
repository found in the session's project directories, at session start and stop,
as `git.snapshot` events.

## Context

DESIGN §5.4 defines the event shape; §7.4 the capture points. Snapshots let the digest
say "this session took repo X from commit A (clean) to commit B (+3 files)" — high
signal for skill synthesis at near-zero cost.

## Scope

- `src/record/files/gitSnapshot.ts`, invoked by the daemon on start (after
  `session.start`) and during graceful stop (before `session.stop`).

## Detailed Requirements

1. Repo discovery: for each project dir, run `git -C <dir> rev-parse --show-toplevel`
   (5 s timeout); non-repos are skipped silently; multiple project dirs resolving to
   the same toplevel are deduplicated.
2. Per repo, gather via separate `execFile` git calls (never shell):
   - `branch`: `git rev-parse --abbrev-ref HEAD` (`HEAD` when detached);
   - `headSha`: `git rev-parse HEAD` (empty repo → `null`, snapshot still emitted);
   - `dirtyFiles`: `git status --porcelain=v1 -z` parsed into
     `[{path, status}]`, capped at 200 entries with `truncated: true` beyond;
   - `diffStat`: `git diff --shortstat HEAD` parsed into
     `{files, insertions, deletions}` (zeros when clean).
3. Emit one `git.snapshot` event per repo with `phase: "start" | "stop"` and
   `repoRoot` relative-ized like file events (`p0:` prefix scheme from issue 14).
4. Failures per repo (git missing, timeout, weird state) → skip with a daemon-log
   entry; git absence entirely → one `session.warning{code:"git-missing"}` only if
   project dirs contain a `.git` directory.
5. Total time budget: all snapshots ≤ 15 s; over-budget repos skipped with log entry
   (stop path must not exceed the daemon drain budget, DESIGN §6).

## Acceptance Criteria

- Tests against generated temp repos: clean repo, dirty repo (staged+unstaged+
  untracked), detached HEAD, empty repo (no commits), non-repo dir, two project dirs
  → one deduped snapshot. Assert exact event `data` shapes.
- `-z` parsing handles paths with spaces and non-ASCII characters (fixture).
- Stop-phase snapshot present after a normal `record stop` (integration with 07 test).

## Validation

`npm test` (git available on both CI OSes).

## Dependencies

04, 07 (+ shares path prefixing with 14).

## Non-goals

Periodic snapshots during the session, full patch capture (the watcher's diffs cover
content), submodule recursion (top-level status only).

## Design References

- DESIGN §5.4, §7.4
