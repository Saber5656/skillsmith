# Title

Consent gate: payload preview, explicit confirmation, consent records

## Summary

Implement `src/consent/`: assembly of the exact backend payload file, the interactive
preview + confirmation flow, the consent record, and the non-interactive `--yes`
policy.

## Context

ADR-004 items 2–3 and DESIGN §10 are normative. This is security boundary B6: nothing
crosses to a backend without a persisted, payload-exact consent.

## Scope

- `src/consent/payload.ts` (payload file assembly given prompt preamble + redacted
  digest), `preview.ts`, `gate.ts`. Invoked only by `generate` (31); no standalone CLI.

## Detailed Requirements

1. `assemblePayload(runDir, {promptPreamble, redactedDigest})`:
   - Concatenate preamble (from 27) + `\n\n` + redacted digest; write
     `generated/<runId>/payload.md` (0600). Return `{path, sha256, chars}`.
   - Verify the redaction report freshness precondition (20): caller passes the
     report; `sourceDigestSha256` must match the digest used — else typed error
     (defense in depth; 31 also checks).
2. `previewAndConfirm({payloadPath, summary, out}): Promise<'granted'|'declined'>`:
   - Print summary table (stderr/human): payload chars, block count, redaction hits by
     rule (id×count), applied reductions, backend id, model (or `backend default`),
     timeout.
   - Open payload in `$PAGER` (default `/usr/bin/less`, argv array); if no TTY on
     stdin/stdout → skip pager, print first 40 lines + `--dry-run` hint.
   - Prompt exactly: `Send this payload to backend "<id>"? [y/N] ` — accept `y`/`yes`
     (case-insensitive) as grant; **everything else declines**.
3. `recordConsent(runDir, {payloadSha256, backend, model, mode})` →
   `consent.json` (0600): `{v:1, payloadSha256, backend, model, consentedAt,
   appVersion, mode: "interactive"|"non-interactive"}`.
4. Non-interactive policy:
   - `--yes` flag honored **only if** `config.consent.allowNonInteractive === true`;
     otherwise exit 5 with message explaining both the flag and the config key
     (ADR-004 double opt-in).
   - No TTY and no `--yes` → exit 5 (never hang waiting for input in scripts).
5. Decline path: exit 5; run directory retained with `payload.md` for inspection; a
   `declined.json` marker `{declinedAt}` prevents accidental reuse of the runId.
6. Consent verification helper for 31: `verifyConsent(runDir, payloadSha256)` —
   consent exists, hash matches, no `declined.json`.

## Acceptance Criteria

- Tests with scripted stdin/PTY stub: `y` → granted + consent.json shape golden;
  `Enter`/`n`/garbage → declined + exit 5 + declined marker; no-TTY without `--yes` →
  exit 5; `--yes` with config false → exit 5; with config true → granted with
  `mode: "non-interactive"`.
- Payload hash in consent matches file bytes (recompute in test).
- Pager invocation uses argv array and falls back cleanly when `$PAGER` is missing
  (subprocess stub).
- Stale redaction report → typed error before any preview.

## Validation

`npm test`; manual interactive run on a recorded session (screenshot of the summary +
prompt in PR).

## Dependencies

03, 05, 06, 20 (27/31 integrate later; for tests use a stub preamble string).

## Non-goals

Payload editing UI (v2), partial consent/per-block exclusion (v2), backend invocation
(22–26).

## Design References

- DESIGN §10, §14.1 B6; ADR-004
