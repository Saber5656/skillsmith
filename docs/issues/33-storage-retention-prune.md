# Title

Storage retention: `sessions prune`

## Summary

Implement retention cleanup for the session store: age- and size-based pruning with
dry-run, never touching the active session, with symlink-safe deletion.

## Context

Screenshots make sessions heavy (DESIGN ADR-005 consequence); doctor (32) warns on
disk usage and points here. Deletion safety rules extend issue 05's `deleteSession`
primitive (boundary B4).

## Scope

- `src/store/prune.ts` + `sessions prune` CLI wiring.

## Detailed Requirements

1. Selection policy (both criteria combinable; defaults from config):
   - age: sessions with state `stopped|failed` whose `stoppedAt` (or `createdAt` for
     `failed`) is older than `--older-than <days>` (default `storage.retentionDays`);
   - size: after age-pruning, if total store size still exceeds
     `--max-total-size <MB>`, delete oldest-first (by session id order) until under.
2. Absolute exclusions: the active session (lock check), sessions in
   `recording|paused|stopping` state regardless of lock (defensive), and any path
   failing 05's inside-store/symlink verification (skip + warn, never force).
3. `--dry-run`: table of candidates `id | state | stoppedAt | sizeMB | reason(age/size)`
   + totals; deletes nothing.
4. Execution output: same table + per-session ok/fail; partial failures don't stop the
   run; summary line with freed MB; `--json` variant with the same data.
5. Size computation: recursive du of each session dir, cached in the run; total
   includes only `sessions/` (not config).
6. No interactive confirmation when criteria flags are explicit; with **no** flags and
   no config retention set → require `--older-than` or `--max-total-size` (exit 2)
   so bare `sessions prune` never surprises.
7. Generated skills in output directories are never touched (only the store).

## Acceptance Criteria

- Tests on a synthetic store (12 sessions, mixed states/ages/sizes, one symlinked
  imposter dir, one active): age policy, size policy, combined policy, exclusions
  (active + recording + symlink skip-with-warning), dry-run zero-mutation (fs
  snapshot), partial-failure continuation (chmod-induced), `--json` schema.
- Bare invocation without criteria exits 2 with guidance.
- Freed-space accounting matches actual du delta (tolerance for fs block sizes).

## Validation

`npm test`; manual dry-run against the developer's real store, output pasted
(redacted) in PR.

## Dependencies

03, 05, 06, 07 (active-session semantics).

## Non-goals

Archival/export before delete (v2 `sessions export`), per-artifact pruning
(screenshots-only cleanup — v2), automatic background pruning (explicit command only
in v1).

## Design References

- DESIGN §13.2, §14.1 B4; ADR-005
