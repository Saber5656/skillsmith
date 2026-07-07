# Research: macOS capture and OCR primitives

- Status: verified locally on macOS 26.5.1 (build 25F80), 2026-07-07
- Method: direct execution of the candidate tools on the development machine
- Impact: grounds ADR-001 (capture methods), ADR-006 (OCR strategy), issues 10–13

## Verified facts (local evidence)

| Claim | Evidence |
|---|---|
| `screencapture` present at `/usr/sbin/screencapture` | `which screencapture` |
| Silent, scriptable capture supported | usage lists `-x` (no sound), `-t<format>` (png default), `-D<display>` (per-display), `-m` (main monitor only), `-C` (include cursor) |
| `sips` present at `/usr/bin/sips` (downscaling) | `which sips` |
| `osascript` JXA with ObjC bridge present | `which osascript` |
| **Vision framework reachable from JXA** | `osascript -l JavaScript -e 'ObjC.import("Vision"); typeof $.VNRecognizeTextRequest'` → available |
| Frontmost app metadata via JXA | `$.NSWorkspace.sharedWorkspace.frontmostApplication` returned name + bundle id |

## Screen capture design inputs

- Invocation shape for the capturer (issue 10):
  `screencapture -x -t png -D <display> <output.png>` per display, on an interval.
- Downscale to cap disk/token cost: `sips --resampleWidth <px> <file>` in place.
- **TCC (Screen Recording permission)**: capture silently produces wallpaper-only or
  fails without the permission, attributed to the *responsible process* (usually the
  terminal app that spawned skillsmith). Consequences:
  - `skillsmith doctor` must detect an unusable capture (test capture + heuristic:
    all-black/wallpaper-only or non-zero exit) rather than assume an API answer.
  - `skillsmith init` must guide the user to
    `x-apple.systempreferences:com.apple.preference.security?Privacy_ScreenCapture`.
  - Known unknown: TCC attribution when the capture loop runs in a detached daemon
    spawned by the CLI. Must be validated on a real machine during issue 10; if the
    daemon is denied while the terminal is granted, fallback is running capture in the
    daemon's process group leader spawned via the user's session (documented in issue 10
    validation steps).

## Window metadata

- Frontmost application name/bundle id: JXA `NSWorkspace` (verified above).
- Window title: `CGWindowListCopyWindowInfo` via JXA ObjC bridge, or Accessibility API
  (requires separate AX permission). v1 uses CGWindowList (no extra permission for
  titles of on-screen windows owned by other apps — titles may be empty for some apps;
  acceptable degradation, event field is optional).

## OCR strategy inputs

Candidates evaluated:

| Option | Extra install | Notes |
|---|---|---|
| Vision via JXA ObjC bridge | none | Class reachable (verified). Runs `VNRecognizeTextRequest` on image files. Performance/accuracy on real screenshots is a **spike** inside issue 12. Supports recognition languages incl. Japanese. |
| Vision via compiled Swift helper | Xcode CLT or shipped binary | Best performance, but shipping a compiled/notarized binary complicates npm supply chain; deferred to v2 unless the JXA spike fails. |
| `tesseract` (Homebrew) | user-installed | Mature fallback; auto-detected on PATH; quality on UI text is lower than Vision for CJK. |
| `shortcuts` CLI with a user Shortcut | manual Shortcut install | Fragile, hard to version; rejected. |

Decision recorded in ADR-006: default `vision-jxa`, fallback `tesseract`, engine
interface keeps both behind one contract, `ocr.engine: auto` picks vision-jxa on macOS.

## Shell hook inputs (for completeness)

- macOS default login shell is zsh; zsh provides `preexec`/`precmd` hook arrays and the
  `zsh/datetime` module (`$EPOCHREALTIME`) so the hook can timestamp without spawning
  processes. bash support is deferred (ADR-001 non-goal for v1).
