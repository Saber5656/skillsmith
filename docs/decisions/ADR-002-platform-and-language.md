# ADR-002: macOS-first, TypeScript/Node.js CLI, npm distribution

- Status: accepted
- Date: 2026-07-07
- Decision maker: repository owner (explicit confirmation), recorded by design agent

## Context

Screen capture and OCR are strongly OS-coupled. A v1 that targets every OS multiplies
the capture matrix; a native (Swift) app maximizes capture quality but minimizes
portability and narrows the OSS contributor pool for an agent-skills-adjacent tool.

## Decision

- v1 targets **macOS only** (macOS 14+; developed against macOS 26).
- Implementation language is **TypeScript** on **Node.js >= 20**, strict mode.
- Distribution via **npm** (`npm install -g skillsmith`), package name `skillsmith`,
  binary `skillsmith`.
- OS-specific operations (screen capture, OCR, frontmost app, settings deep links) are
  isolated behind small adapter interfaces in dedicated modules so a Linux port in v2
  changes adapters, not the pipeline.
- No compiled native code is shipped in v1; all macOS integration goes through
  OS-bundled executables (`screencapture`, `sips`, `osascript`) — see
  `docs/research/macos-capture-and-ocr.md`.

## Consequences

- Fast development, single toolchain, easy CI on GitHub's macOS runners.
- Node.js is a runtime prerequisite (acceptable: target users run agent CLIs that are
  themselves Node- or npm-adjacent).
- Screenshot/OCR throughput is bounded by subprocess spawning; acceptable at the v1
  default interval (10 s).
- Linux/Windows users are v2; the README must state this clearly.

## Alternatives rejected

- Rust single binary: better distribution story, slower iteration, worse ecosystem fit.
- Cross-platform v1: doubles capture/test matrix before product validation.
- Swift menubar app: best capture UX, worst portability and OSS reach; CLI-first fits
  the target persona (engineers automating agent workflows).
