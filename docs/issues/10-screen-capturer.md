# Title

Screen capturer: interval screenshots, frontmost-app metadata, denylist, TCC handling

## Summary

Implement the screen recorder: a timer loop that captures PNG screenshots via
`screencapture`, downscales via `sips`, tags frontmost app/window metadata via JXA,
skips denylisted apps, and detects missing Screen Recording permission.

## Context

DESIGN §7.2 is normative; local tool evidence in `docs/research/macos-capture-and-ocr.md`.
This recorder feeds the OCR queue (11). The TCC-in-daemon question is a tracked known
unknown — this issue must resolve it on real hardware.

## Scope

- `src/record/screen/capturer.ts`, `frontmost.ts`, `denylist.ts` (Recorder registration
  per 07). OCR itself is 11–13.

## Detailed Requirements

1. Loop every `recording.screenshot.intervalSeconds` (config), skewed ±10 % to avoid
   phase-locking with other periodic work. Loop pauses in `paused` state.
2. Per tick:
   a. Frontmost app via JXA (single `osascript -l JavaScript` invocation returning
      JSON: `{name, bundleId}`; script asset `assets/jxa/frontmost.js`). Window title
      best-effort via CGWindowList in the same script; `null` on failure. Timeout 5 s.
   b. If `bundleId ∈ recording.screenshot.appDenylist` → emit
      `screen.skipped{reason:"denylist", frontmostApp}` and return.
   c. Capture: `captureAllDisplays: true` → enumerate displays by capturing
      `screencapture -x -t png <f1> <f2> …` (one file per display, single invocation);
      else `-m` main only. Files: `screens/<seq>-d<display>.png` with zero-padded
      global capture seq.
   d. Downscale each file in place: `sips --resampleWidth <maxWidthPx>` only when wider.
   e. Emit `screen.capture` per display file with metadata; enqueue for OCR (11).
3. TCC/permission failure detection: `screencapture` non-zero exit, missing output
   file, or output < 8 KiB (wallpaper-only heuristic is unreliable; use the documented
   triple) → emit `screen.skipped{reason:"error"}` and, once per session,
   `session.warning{code:"screen-permission", message: <fix hint>}`; after 3 consecutive
   failures set recorder state `error` (visible in `record status`) and stop the loop.
4. All subprocess calls: absolute paths (`/usr/sbin/screencapture`, `/usr/bin/sips`,
   `/usr/bin/osascript`), argv arrays, 15 s timeouts, stderr captured to daemon log (B2).
5. Disk guard: before each capture, if session `screens/` exceeds 80 % of
   `maxSessionSizeMB` budget share, emit warning once and increase effective interval
   ×4 (do not silently stop).
6. **Known-unknown resolution (mandatory)**: document in the PR the observed TCC
   behavior when the daemon (spawned detached from the CLI under Terminal/iTerm) calls
   `screencapture` — granted via the parent app's permission or denied. If denied:
   implement and document the fallback of running the capture loop in a
   non-detached child owned by the CLI process group, and record the finding in
   `docs/research/macos-capture-and-ocr.md` + a follow-up note in ISSUE_PLAN known
   unknowns.

## Acceptance Criteria

- Unit tests with a stubbed subprocess layer: denylist skip path, failure escalation
  (3 strikes → recorder error), downscale-only-when-wider, seq numbering with
  multi-display, interval skew bounds, pause behavior.
- mac-local test (`test:local`): 30 s recording produces ≥ 2 `screen.capture` events
  with real PNGs, correct metadata fields present, files ≤ maxWidthPx wide.
- Permission-denied simulation (subprocess stub): warning emitted once, recorder state
  `error` after 3 failures, daemon keeps running other recorders.
- PR includes the TCC finding write-up (req 6).

## Validation

`npm test` + `npm run test:local` on real hardware with screenshots inspected; attach a
redacted `record status --json` sample.

## Dependencies

03, 04, 05, 07.

## Non-goals

OCR (11–13), video, per-window capture, screen-lock detection (v2), Linux adapters (v2).

## Design References

- DESIGN §7.2, §14.1 B2; ADR-001; docs/research/macos-capture-and-ocr.md
