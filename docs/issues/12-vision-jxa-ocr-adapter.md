# Title

Vision OCR adapter via JXA ObjC bridge (default engine) — includes feasibility spike gate

## Summary

Implement the `vision-jxa` OCR engine: a bundled JXA script calling Apple Vision's
`VNRecognizeTextRequest` through `osascript`, plus the adapter that invokes it in
batches. Contains an explicit spike gate that can flip the default engine per ADR-006.

## Context

Local verification confirmed the JXA ObjC bridge reaches Vision classes
(`docs/research/macos-capture-and-ocr.md`). Unproven: real-world latency and accuracy.
ADR-006 defines the fallback decision procedure.

## Scope

- `assets/jxa/ocr.js` (JXA script), `src/record/ocr/visionJxa.ts` (adapter).

## Detailed Requirements

1. JXA script contract (stdin/argv → stdout JSON):
   - argv: image file paths (batch); env `SKILLSMITH_OCR_LANGS` comma-separated.
   - For each image: load via `NSURL`/`VNImageRequestHandler`, run
     `VNRecognizeTextRequest` with `recognitionLevel: accurate`,
     `recognitionLanguages` from env, `usesLanguageCorrection: true`.
   - stdout: one JSON document
     `[{file, status, lines: [{text, confidence, bbox: [x,y,w,h]}], durationMs, error?}]`
     (bbox in normalized coordinates as provided by Vision; pass through).
   - Per-image try/catch: one bad image must not fail the batch.
2. Adapter:
   - `detect()`: run the script with zero args + a `--probe` flag that only imports
     Vision and prints `{"probe":"ok"}`; 10 s timeout.
   - `recognize(batch, opts)`: single `osascript -l JavaScript assets/jxa/ocr.js
     <files…>` invocation (absolute `/usr/bin/osascript`), timeout from opts, stdout
     cap 20 MiB, JSON parse with schema validation; map failures per file.
   - Language mapping: config `ocr.languages` (e.g. `[en, ja]`) → Vision identifiers
     (`en-US`, `ja-JP`) via a fixed table; unknown entries warned and dropped.
3. **Spike gate (blocking acceptance)**: measure on real hardware with 5 fixture
   screenshots (code editor, terminal, browser article, Japanese UI text, mixed
   dashboard — commit fixtures ≤ 300 KiB each under `test/fixtures/screens/`):
   - Record per-image latency and a subjective accuracy check (expected key phrases
     present in output; fixture-specific assertion lists).
   - Pass condition: p95 latency ≤ 10 s/image and all fixture key-phrase assertions
     pass. On failure: per ADR-006, flip `auto` default order to prefer `tesseract`,
     update ADR-006 status note + `docs/research/macos-capture-and-ocr.md`, and open a
     v2 issue for a Swift helper. The adapter still ships either way.

## Acceptance Criteria

- `detect()` true on a real mac; false (not crash) when `osascript` is stubbed to fail.
- Fixture batch returns non-empty lines with confidences for the English fixtures and
  the Japanese fixture (key-phrase assertions green, mac-only test).
- Malformed image in a batch → its entry `status:"failed"`, others `ok`.
- Spike results table committed into the PR description and reflected per ADR-006.
- Script asset resolves from a global npm install (no cwd assumptions).

## Validation

`npm run test:local` on real hardware (spike); CI macos job runs the fixture test if
Vision works on runners — if runners lack OCR capability, mark test `mac-local` and CI
skips with an explicit skip reason (never silently green).

## Dependencies

11.

## Non-goals

Image preprocessing, bounding-box-based layout reconstruction (17 uses plain lines),
Swift helper (v2).

## Design References

- DESIGN §7.3; ADR-006; docs/research/macos-capture-and-ocr.md
