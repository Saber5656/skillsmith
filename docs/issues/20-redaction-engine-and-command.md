# Title

Redaction engine, report, and `skillsmith redact` command

## Summary

Implement `src/redact/engine.ts` + `report.ts` and the `redact` CLI command: apply the
rule library to a digest with stable placeholders, produce the redaction report, wire
custom patterns/allowlist from config, and enforce fail-closed semantics.

## Context

ADR-004 items 1, 4, 5 and DESIGN §9.2 are normative. `generate` (31) consumes this via
the digest→redacted-digest path; the log sanitizer hook from issue 06 is also completed
here.

## Scope

- Engine + report + `redact <id> [--dry-run] [--json]` command + logger sanitizer
  wiring. Input is the digest markdown (18) — field-level application happens by
  redacting the digest as a whole plus the note/marker inputs before they enter it.

## Detailed Requirements

1. `redactText(text, {rules, customPatterns, allowlist}): {text, hits}`:
   - Single pass per rule in library order (19); custom patterns run after built-ins
     with severity `high`.
   - Allowlist: exact-string matches are protected (replaced by placeholders-for-scan,
     restored after) — allowlisted strings never masked, never entropy-flagged.
   - Masking: replace the matched secret span with `[REDACTED:<rule-id>:<n>]`;
     for `url-credentials` mask only the password capture group; for `pem-block`
     mask the whole block. `<n>`: per-run stable — identical original values get the
     same n (map value→n, first-seen order, per rule id).
   - Overlap policy: earlier-rule match wins; later rules skip already-masked spans.
2. `redactSession(sessionId, config)`:
   - Load session (must be `stopped`), build timeline → blocks → distilled → digest
     (16–18) with `maxChars` from config.
   - Redact the digest text; write `redacted/digest.redacted.md` (0600) and
     `redacted/report.json`:
     `{v:1, generatedAt, sourceDigestSha256, ruleHits: [{ruleId, count}],
     allowlistHits, appliedReductions, engineVersion}`.
   - **Fail-closed**: any thrown error (rule exec budget exceeded, invalid custom
     regex at runtime, I/O) → delete partial outputs, rethrow typed `RedactionError`;
     `generate` (31) requires a report whose `sourceDigestSha256` matches the current
     digest.
3. CLI `redact <id>`:
   - Human output: table of rule hits (id, severity, count) + output paths;
     `--dry-run`: print report to stdout, write nothing; `--json`: report JSON.
   - Exit 3 if session not `stopped`/not found; exit 1 on `RedactionError`.
4. Logger sanitizer (closes issue 06's TODO): free-text log arguments pass through
   `HIGH_SEVERITY_RULES` masking; wire into the shared logger construction.
5. Placeholder collision guard: input text already containing
   `[REDACTED:` is escaped (`[REDACTED-LITERAL:`) before scanning so reports can't be
   spoofed by recorded content (B5 integrity).

## Acceptance Criteria

- Engine tests: stable numbering (same key twice → same n); overlap policy; URL
  password-only masking; allowlist protection incl. entropy exemption; custom pattern
  application; placeholder collision guard.
- Session-level test on a fixture session seeded with 10 synthetic secrets across
  shell/OCR/diff/note fields → all masked in `digest.redacted.md`, report counts match
  exactly, byte-stable across two runs.
- Fail-closed test: custom pattern that exceeds the execution budget → no
  `redacted/` outputs remain, exit 1, error names the pattern id.
- Log sanitizer test: logging a string containing a synthetic `AKIA…` key produces a
  masked log line.

## Validation

`npm test`; manual `redact --dry-run` on a locally recorded session, output pasted
(redacted) into PR.

## Dependencies

03, 05, 06, 16, 17, 18, 19.

## Non-goals

Redacting raw event files in place (originals immutable, ADR-004), image/pixel
redaction (v2), consent UX (21).

## Design References

- DESIGN §9.2, §14.1 B5; ADR-004
