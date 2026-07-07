# Title

Screen text distillation: OCR dedup and window-title tracking

## Summary

Implement `src/timeline/distill.ts`: collapse near-duplicate OCR text across
consecutive captures into delta-only text per activity block, and produce a compact
window-title timeline.

## Context

Screenshots every 10 s of a mostly-static screen produce massively redundant OCR text;
without dedup the digest would blow its budget on repetition. DESIGN §8.2 is normative.

## Scope

- Pure module consuming Block objects (16): `distillScreenText(block, opts): DistilledScreen`.

## Detailed Requirements

1. Line normalization for comparison: trim, collapse internal whitespace, NFC,
   lowercase. Comparison only — emitted text keeps original form.
2. Delta extraction: for each capture in block order (per display id independently),
   emit only lines whose normalized form was **not** present in the previous capture of
   the same display (set-based, order-preserving). First capture of a block emits all
   lines.
3. Confidence floor: lines with `confidence < 0.35` are dropped before delta logic
   (tesseract lines without confidence pass through).
4. Per-block cap: retained screen text ≤ `capChars` (default 4000). Reduction order:
   drop lowest-confidence lines first, then oldest captures' deltas first; record
   `truncated: true` and dropped counts.
5. Window-title timeline: ordered unique-run-length list
   `[{title, bundleId, fromTs, toTs}]` merged across captures (consecutive identical
   title+bundle collapse); titles deduped case-sensitively; missing titles → app name.
6. Output shape:
   `{deltaText: [{ts, displayId, lines: string[]}], titles: [...], stats: {rawLines,
   emittedLines, droppedLowConf, truncated}}`.
7. Determinism as in issue 16.

## Acceptance Criteria

- Fixture: 6 captures of a mostly-static terminal with 2 changed regions → deltas
  contain only changed lines (golden).
- Confidence floor and cap behavior verified with synthetic OCR line sets, including
  the reduction order (lowest-conf-first proven by construction).
- Title run-length merge verified across app switches.
- Property-style test: emitting all first-capture lines + deltas never exceeds the raw
  union of lines (no duplication or invention).

## Validation

`npm test` (platform-neutral).

## Dependencies

16 (Block shape), 11 (OcrLine shape).

## Non-goals

Layout reconstruction from bboxes, cross-block dedup (blocks are independent by
design), semantic summarization (LLM's job).

## Design References

- DESIGN §8.2
