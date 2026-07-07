# ADR-006: OCR via Apple Vision through JXA, with tesseract fallback

- Status: accepted (with an explicit feasibility spike gate)
- Date: 2026-07-07
- Decision maker: design agent (grounded in local verification)

## Context

Screen understanding in v1 is OCR-based (ADR-001). The OCR engine must work with zero
extra installs for a good first-run experience, handle CJK text (target users include
Japanese-speaking engineers), and avoid shipping compiled binaries in the npm package
(supply-chain simplicity).

## Decision

- Define a single `OcrEngine` interface with pluggable adapters; OCR runs as an async
  queue inside the recording daemon, decoupled from the capture loop.
- Default engine `vision-jxa`: a bundled JXA (JavaScript for Automation) script run via
  `/usr/bin/osascript`, using the ObjC bridge to call Vision's `VNRecognizeTextRequest`
  on captured PNGs. Local verification confirms the bridge reaches the Vision classes
  (`docs/research/macos-capture-and-ocr.md`).
- Fallback engine `tesseract`: used when detected on `PATH` and selected by config or by
  `auto` resolution when vision-jxa is unavailable/broken.
- `ocr.engine: auto | vision-jxa | tesseract | off` in config; `off` degrades screen
  events to app/window metadata only.
- **Spike gate inside issue 12**: before completing the adapter, measure recognition
  latency and accuracy on 5 representative screenshots (code editor, terminal, browser,
  Japanese text, mixed). If p95 latency per image exceeds 10 s or accuracy is unusable,
  record the result here, flip the default to `tesseract`, and open a v2 issue for a
  compiled Swift helper.

## Consequences

- Zero-install OCR by default; no notarization or binary shipping in v1.
- osascript spawning per image adds overhead — bounded by the queue and the 10 s
  capture interval; the queue may batch multiple images per osascript invocation.
- OCR quality becomes a documented, testable property (fixture corpus in issue 35).

## Alternatives rejected

- Compiled Swift helper in v1: best performance, but adds binary distribution,
  signing/notarization, and supply-chain weight; deferred to v2 behind the same interface.
- `shortcuts` CLI: unversionable manual setup; rejected.
- tesseract as default: forces a Homebrew install on first run and is weaker on CJK UI text.
