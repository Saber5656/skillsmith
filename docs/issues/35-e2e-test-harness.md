# Title

End-to-end test harness: fixture session, fake backend, golden pipeline

## Summary

Build the E2E test infrastructure: a complete canned session fixture, the
deterministic fake backend script, golden outputs for the full
`redact → generate → validate` pipeline, CI wiring, and the local-only recorder smoke
suite (`npm run test:local`).

## Context

DESIGN §15.2 defines the testing strategy. Unit/integration tests live with their
issues; this issue delivers the shared fixtures, the full-pipeline goldens, and the
split between CI-runnable and real-hardware suites.

## Scope

- `test/fixtures/sessions/full-01/` (complete session dir), `test/fixtures/bin/fake-backend.mjs`,
  `test/e2e/pipeline.e2e.test.ts`, `test/local/recorders.local.test.ts`, npm script +
  CI wiring updates.

## Detailed Requirements

1. Fixture session `full-01` (committed, hand-authored, total ≤ 2 MiB):
   - realistic `manifest.json` (stopped), `events.jsonl` (~200 events: a plausible
     "deploy a service" storyline across 3 blocks with idle gap, incl. markers,
     pause/resume, one denylist skip, one failed OCR), 4 tiny PNGs (≤ 50 KiB each),
     matching `ocr/*.json` (canned text incl. Japanese lines and 3 planted synthetic
     secrets), `diffs/*.patch`, git snapshots. Planted secrets are documented in a
     `SECRETS-PLANTED.md` inside the fixture (consumed by 34's B5 audit).
2. Fake backend `fake-backend.mjs` (node script, no deps):
   - reads stdin fully; sha256-logs invocation to `$FAKE_BACKEND_LOG` if set (consent
     ordering assertions, 31); emits a deterministic sentinel response derived from a
     canned template — flags via env: `FAKE_MODE=happy|bad-frontmatter|hostile-paths|
     low-confidence|garbage|slow` (slow: sleep 400 s for timeout tests).
   - registered in tests through `backend.custom` config (26).
3. E2E test (CI, macOS + Linux — fully platform-neutral by construction):
   - temp HOME + XDG; copy fixture; run built CLI: `redact` → assert report goldens;
     `generate --yes` with `allowNonInteractive: true` test config → assert skill dir
     goldens (byte-stable), validator green, provenance complete, planted secrets
     absent everywhere (B5 quick check; deep audit in 34);
   - each FAKE_MODE failure path asserted (exit codes 6/7, artifacts per 31).
4. Local suite (`test:local`, real hardware only, clearly separated):
   - records a real 45 s session in a scratch project dir (scripted zsh commands +
     file edits), asserts event counts ≥ expectations, then runs the full pipeline
     with the fake backend. Requires screen permission; prints skip-with-reason
     otherwise (never false-green).
5. Goldens under `test/golden/` with an `UPDATE_GOLDEN=1` refresh mechanism and a
   review note in CONTRIBUTING (36).
6. CI: E2E job added to the workflow (02) on both OSes; `test:local` documented as a
   release-gate manual step (37 checklist).

## Acceptance Criteria

- E2E green on both CI OSes from a clean checkout (`npm ci && npm run build && npm
  test`).
- Determinism: two consecutive E2E runs produce identical goldens (hash compare in
  test).
- Fake backend modes each exercised; consent-ordering log assertion present.
- Local suite runs on the developer machine with all assertions green — output
  attached to PR.
- Fixture documented (`full-01/README.md`: storyline, event inventory, planted
  secrets pointer).

## Validation

CI runs + attached local-run evidence.

## Dependencies

26, 29, 31 (pipeline complete); fixtures consumed by 34.

## Non-goals

Performance benchmarking, fuzzing, multi-version backend matrix (adapters pin their
own verification).

## Design References

- DESIGN §15.2; ADR-004 (secret-free assertion), ADR-007 (hostile-path mode)
