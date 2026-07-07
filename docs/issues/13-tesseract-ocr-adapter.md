# Title

Tesseract OCR adapter (fallback engine)

## Summary

Implement the `tesseract` OCR engine adapter: detection of a user-installed tesseract
binary, batch invocation, TSV output parsing into the common `OcrLine` shape.

## Context

ADR-006 designates tesseract as the fallback when Vision-via-JXA is unavailable or
fails its spike gate. Users install it themselves (`brew install tesseract`); skillsmith
never bundles it.

## Scope

- `src/record/ocr/tesseract.ts` implementing `OcrEngine` (interface from 11).

## Detailed Requirements

1. `detect()`: locate `tesseract` on `PATH` (`which` semantics via `execFile`), run
   `tesseract --version` (5 s timeout), parse major version (require ≥ 4); report
   available languages via `tesseract --list-langs` and cache the list.
2. Language mapping: config `ocr.languages` `[en, ja]` → tesseract codes
   (`eng`, `jpn`) via fixed table; requested-but-uninstalled languages → one-time
   warning (`session.warning{code:"ocr-lang-missing"}`), proceed with the installed
   subset; empty subset → `eng`.
3. `recognize(batch, opts)`: per image (tesseract has no multi-image batch):
   `tesseract <img> stdout -l <langs> --psm 3 tsv` with per-image timeout
   `opts.timeoutMs / batch.length` (min 5 s); parse TSV rows (level 5 = word) →
   reassemble lines by (block, par, line) keys, joining words with spaces, line
   confidence = mean word conf / 100; skip conf −1 rows.
4. Whitespace/CJK join rule: when all words in a line are CJK, join without spaces.
5. Errors: non-zero exit / timeout / unparsable TSV → that image `status:"failed"`
   with `error` message; never throw batch-wide.
6. Output shape identical to 11's `OcrLine` (no bbox in v1 for tesseract — omit field).

## Acceptance Criteria

- Unit tests with canned TSV fixtures: line reassembly, CJK join rule, conf −1
  skipping, mixed failure batch.
- `detect()` matrix: missing binary, old version (stub), healthy (mac-local with real
  tesseract if present — skip with reason otherwise).
- Same 5 fixture screenshots as issue 12 produce non-empty output for English fixtures
  (accuracy assertions relaxed vs Vision; document expected weaker CJK results).

## Validation

`npm test` (fixture parsing paths); `npm run test:local` when tesseract installed.

## Dependencies

11 (interface), 12 (shared fixture set).

## Non-goals

Bundling tesseract, tessdata management, PSM tuning per app (v2).

## Design References

- DESIGN §7.3; ADR-006
