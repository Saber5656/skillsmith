# Title

OCR engine interface and asynchronous OCR queue

## Summary

Define the `OcrEngine` interface and implement the bounded in-daemon OCR queue that
processes captured screenshots asynchronously and emits `screen.ocr` events.

## Context

DESIGN §7.3 is normative; ADR-006 fixes the engine strategy (adapters in 12/13).
Decoupling capture from OCR keeps the capture cadence stable even when OCR is slow.

## Scope

- `src/record/ocr/engine.ts` (interface + registry + `auto` resolution),
  `src/record/ocr/queue.ts`. Adapters are issues 12 and 13.

## Detailed Requirements

1. Interface:
   ```ts
   interface OcrEngine {
     id: 'vision-jxa' | 'tesseract';
     detect(): Promise<{ available: boolean; detail?: string }>;
     recognize(batch: { file: string }[], opts: { languages: string[]; timeoutMs: number }):
       Promise<{ file: string; status: 'ok' | 'failed'; lines: OcrLine[]; durationMs: number; error?: string }[]>;
   }
   type OcrLine = { text: string; confidence?: number; bbox?: [x, y, w, h] };
   ```
2. `resolveEngine(config)`: `off` → null; explicit id → that engine (error if
   `detect()` fails at daemon start → recorder state `error`, capture continues without
   OCR); `auto` → first available of `vision-jxa`, `tesseract`, else `off` with a
   one-time `session.warning{code:"ocr-unavailable"}`.
3. Queue:
   - `enqueue(captureEvent)` called by the capturer; bounded at 32 entries; overflow
     drops the **oldest** pending entry and increments a dropped counter (surfaced in
     heartbeat counters + one warning event per session).
   - Worker loop: dequeue up to 4 entries → single `recognize` batch call
     (`timeoutMs: 30_000`) → write each result to `ocr/<seq>-d<display>.json`
     (`{engine, lines, durationMs}` mode 0600) → emit `screen.ocr` with
     `captureRef, file, engine, status, textLength, durationMs`.
   - Failures: engine batch throw/timeout → each entry gets `status:"failed"` event
     (no result file); 5 consecutive failed batches → engine disabled for the session
     (warning event, capture continues).
   - Drain on stop: process remaining queue up to the daemon's 30 s drain budget
     (DESIGN §6); leftovers are dropped with a final count in `session.stop` handling
     (log only).
4. Text normalization before storage: strip zero-width chars, collapse >2 consecutive
   blank lines; preserve original line order and confidence.

## Acceptance Criteria

- Unit tests with a mock engine: batching (4 max), overflow drop-oldest, failure
  escalation (5 strikes), drain-on-stop within budget, result file shape golden.
- `auto` resolution matrix test (vision available / only tesseract / neither).
- Queue never blocks `enqueue` (measured: enqueue returns < 1 ms with a stalled worker).

## Validation

`npm test`; integration with real adapters validated in 12/13 and `test:local`.

## Dependencies

03, 04, 07, 10.

## Non-goals

Concrete engines (12, 13), OCR text dedup/distillation (17), image preprocessing (v2).

## Design References

- DESIGN §7.3; ADR-006
