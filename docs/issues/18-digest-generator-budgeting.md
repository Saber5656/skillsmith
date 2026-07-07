# Title

Digest generator with deterministic token budgeting

## Summary

Implement `src/timeline/digest.ts`: render the segmented, distilled timeline into the
canonical Markdown digest with a deterministic size-reduction ladder that fits
`generate.maxPayloadChars`.

## Context

The digest is the *only* session representation the LLM ever sees, and (after
redaction) the exact content the user consents to. DESIGN §8.3 fixes the format and
the reduction order.

## Scope

- `renderDigest(session, blocks, opts): {markdown, meta}`; consumed by `redact` (20)
  and `generate` (31).

## Detailed Requirements

1. Structure (exact section order, DESIGN §8.3):
   - Header: `# Session digest` + meta table (session id, date, duration, note,
     project dirs, counters, `corruptLines` if > 0) + `truncation:` list (empty →
     `none`).
   - Per block: `## Block N — HH:MM–HH:MM — <dominant project or app>` with
     subsections `### Markers`, `### Shell`, `### File changes`, `### Screen` (each
     omitted when empty).
2. Rendering rules:
   - Shell: fenced block, one line per command: `<cwd> $ <command>  # exit <code>,
     <duration>`; consecutive identical commands collapsed with `×N`.
   - File changes: table `path | change | ±lines`; then the diffs of the 5 most-changed
     text files (by diff size) inline in fenced `diff` blocks.
   - Screen: title timeline line, then delta text grouped by capture timestamp.
   - Timestamps in session-local time HH:MM:SS; all text passed through **verbatim**
     (redaction happens downstream — the digest itself is a sensitive artifact and is
     written only under `redacted/` after issue 20; this module returns strings only
     and must not write files).
3. Budgeting (`maxChars` opt): estimate = `markdown.length`. While over budget, apply
   ladder steps in order, re-render, record each applied step in `truncation:`:
   1. remove Screen sections of blocks that have shell or file activity;
   2. truncate every inline diff to its `@@` hunk headers;
   3. remove all remaining Screen sections;
   4. summarize oldest blocks (all but the newest 10) to a one-line counters row.
   If still over budget after step 4 → hard-truncate tail with a final marker line
   (`meta.hardTruncated = true`).
4. `meta` return: `{chars, appliedReductions[], blockCount, hardTruncated}`.
5. Determinism: identical inputs → identical bytes (no wall clock, stable sorts).

## Acceptance Criteria

- Golden digests for 3 fixture sessions (linear, multi-block, screen-heavy) committed
  under `test/fixtures/digests/` — byte-exact comparison.
- Budget ladder test: shrink `maxChars` stepwise and assert each ladder step engages in
  documented order, `truncation:` lists them, and output ≤ budget at every step.
- Collapse rule (`×N`) and top-5-diff selection verified.
- No file writes (fs spy).

## Validation

`npm test`; golden diffs reviewed by a human in PR.

## Dependencies

16, 17.

## Non-goals

Redaction (19/20), payload preamble/prompt (27), non-Markdown digest formats (v2).

## Design References

- DESIGN §8.3
