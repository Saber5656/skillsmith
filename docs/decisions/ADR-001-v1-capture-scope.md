# ADR-001: v1 capture scope and methods

- Status: accepted
- Date: 2026-07-07
- Decision maker: repository owner (explicit confirmation), recorded by design agent

## Context

The product promise is "generate SKILL.md files from recorded work sessions across
screen, shell, and editor activity". Capture fidelity drives effort, privacy exposure,
and platform coupling. Options ranged from ingest-only (no recorders) to full-fidelity
capture (screen video, PTY wrapper, editor extensions).

## Decision

v1 implements all three capture sources with pragmatic methods:

| Source | v1 method | Explicitly NOT in v1 |
|---|---|---|
| Shell | zsh `preexec`/`precmd` hook appending events to a session ingest file | PTY wrapper, full terminal output capture, bash/fish hooks |
| Screen | periodic still screenshots via `/usr/sbin/screencapture` + OCR text extraction + frontmost-app/window metadata | video recording, VLM image analysis, per-window capture |
| Editor/files | filesystem watcher over declared project directories with bounded text diffs + git snapshots at session start/stop | editor plugins (VS Code/JetBrains), keystroke capture, semantic editor events |

## Consequences

- The full README promise is covered in v1 while avoiding the largest engineering and
  privacy costs (video pipelines, PTY correctness, plugin distribution).
- Screenshots still capture whatever is on screen → the mandatory redaction and consent
  pipeline (ADR-004) plus a frontmost-app denylist are hard requirements, not options.
- Shell capture is command-level, not output-level: generated skills describe *what was
  run*, not full terminal transcripts. Command output can be partially recovered via OCR
  of terminal windows.
- bash/fish hooks, PTY capture, and editor extensions become well-defined v2 items.

## Alternatives rejected

- Shell+files only: breaks the README's core promise ("screen").
- Full fidelity (video + PTY + extensions): multi-quarter scope, heaviest privacy surface.
- Ingest-only: no recorder = no differentiated product.
