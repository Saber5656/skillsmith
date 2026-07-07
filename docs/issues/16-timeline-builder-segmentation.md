# Title

Timeline builder and activity-block segmentation

## Summary

Implement `src/timeline/builder.ts` + `segmenter.ts`: read a stopped session's
`events.jsonl` into a validated, ordered timeline and segment it into activity blocks
with per-block statistics.

## Context

DESIGN §8.1 is normative. The timeline is the neutral intermediate between raw events
and the digest (18); distillation of screen text is issue 17.

## Scope

- Pure, side-effect-free module: `buildTimeline(sessionDir): Timeline` and
  `segment(timeline, opts): Block[]`. No CLI surface.

## Detailed Requirements

1. `buildTimeline`:
   - Load events via `readEvents` (04); expose `corruptLines` on the result.
   - Sort by `(ts, seq)`; join `screen.ocr` results to their `screen.capture` by
     `captureRef` (missing/failed OCR → capture retains `ocr: null`); load OCR line
     files lazily via accessor functions, not eagerly into memory.
   - Compute session stats: duration, counts per kind, distinct apps (from captures),
     distinct cwds, project dirs.
2. `segment(timeline, {idleGapMinutes = 5})`:
   - **User-activity events** = `shell.command`, `file.change`, `session.marker`.
   - New block when the gap between consecutive user-activity events >
     `idleGapMinutes`, or when the dominant project prefix (`p0:`/`p1:`… from cwd and
     file paths) changes across a gap ≥ 1 min.
   - Screen events attach to the block covering their timestamp; captures during idle
     gaps attach to the *following* block (context for what came next); trailing idle
     captures → last block.
   - Pause/resume events close and reopen blocks (a paused span is never inside a
     block).
3. Block shape:
   `{index, startTs, endTs, dominantProject?, dominantApps: [{bundleId, name, captureCount}],
   markers[], shellCommands[], fileChanges[], captures[], warnings[]}` — arrays are
   references into the timeline (no copies).
4. Determinism: same input file → byte-identical JSON serialization of blocks
   (stable ordering everywhere; no wall-clock reads).
5. Guardrails: sessions with 0 user-activity events yield a single block spanning the
   session; > 500 blocks → merge smallest-adjacent until ≤ 500 (record `merged: true`).

## Acceptance Criteria

- Fixture sessions (hand-authored `events.jsonl` committed under
  `test/fixtures/sessions/`): simple linear session; session with a 10 min idle gap →
  2 blocks; project switch → split; pause/resume → block boundaries; OCR-failed capture
  handling; corrupt-line session still builds.
- Determinism test: build twice, deep-equal + stable serialization hash.
- Memory: OCR text not loaded until accessor invoked (spy-based test).

## Validation

`npm test` (platform-neutral; runs on both CI OSes).

## Dependencies

04, 05 (fixtures only need file layout, not live recording).

## Non-goals

Rendering/markdown (18), OCR dedup (17), semantic labeling of blocks (the LLM's job).

## Design References

- DESIGN §8.1
