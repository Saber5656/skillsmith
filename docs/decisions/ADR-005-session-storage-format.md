# ADR-005: session storage — JSONL event log + manifest state machine under XDG paths

- Status: accepted
- Date: 2026-07-07
- Decision maker: design agent (conservative default; no product-scope impact)

## Context

Recorders produce a high-rate, append-only stream from multiple writers (daemon loops +
shell hook). Sessions must survive crashes, be inspectable with standard tools, and be
independently redactable/regeneratable.

## Decision

- Data root: `$XDG_DATA_HOME/skillsmith` (default `~/.local/share/skillsmith`);
  config root: `$XDG_CONFIG_HOME/skillsmith` (default `~/.config/skillsmith`).
  XDG variables are honored on macOS for predictability of a CLI tool.
- One directory per session: `sessions/<session-id>/` with
  `manifest.json` (metadata + lifecycle state), `events.jsonl` (single normalized
  append-only event log written **only by the daemon**), `ingest/` (raw drop files from
  the shell hook), `screens/`, `ocr/`, `diffs/`, `redacted/`, `generated/<runId>/`,
  `daemon.log`.
- Session id: `ss-<YYYYMMDD>-<HHmmss>-<4 hex>` (sortable, unique enough locally).
- Events: JSON Lines with a versioned envelope
  `{v, ts, seq, source, kind, data}`; readers tolerate and count corrupt lines
  (crash-truncated tails) instead of failing.
- Manifest lifecycle states: `created → recording ⇄ paused → stopping → stopped`,
  terminal `failed`; crash recovery finalizes an orphaned session to `stopped` with
  `stopReason: "crash-recovered"`. Derived artifacts (`redacted/`, `generated/`) are
  tracked as artifacts, not lifecycle states, because they are repeatable.
- Permissions: directories `0700`, files `0600`, enforced at creation and audited by
  `skillsmith doctor`.
- A single global `active-session.json` pointer (plus lock file) enforces at most one
  recording session per machine in v1.

## Consequences

- Everything is inspectable with `jq`/`less`; no database dependency.
- Multi-writer safety is achieved by giving the hook its own `ingest/` files (append-only,
  one per shell instance) that the daemon normalizes into the single `events.jsonl`.
- Storage growth is real (screenshots dominate) → retention/prune is a v1 issue, and
  screenshot downscaling is on by default.

## Alternatives rejected

- SQLite store: better queries, worse transparency and crash-forensics for v1; can be
  added later behind the store API.
- macOS-native `~/Library/Application Support`: less predictable for CLI users and
  scripts than XDG; revisit if a GUI ships.
