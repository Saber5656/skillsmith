# Title

Security hardening pass: cross-cutting audit against the threat model

## Summary

Verify and close gaps against every boundary and abuse case in DESIGN §14 across the
implemented codebase: permissions, symlink defenses, parser robustness, subprocess
hygiene, and log sanitization — with regression tests that pin each property.

## Context

Individual issues each carry their local security requirements; this issue is the
adversarial re-check after integration, so drift between issues cannot silently void a
boundary. It is intentionally scheduled after the pipeline works end-to-end.

## Scope

- Audit + fixes + tests. No new features. Findings that require design changes get
  filed as new issues and linked in ISSUE_PLAN known-unknowns.

## Detailed Requirements

Produce `docs/research/security-audit-v1.md` with a checklist row per item below
(status, evidence path — test name or code location, notes):

1. **B1 hook**: source the real hook in a hostile HOME (pointer file replaced by a
   symlink to `/etc/passwd`, pointer JSON with a session id containing `../`) —
   assert no write outside the store, no crash, hook stays silent. Add the missing
   guards if any (session id regex re-validation inside the daemon's ingest path).
2. **B2/B7 subprocess inventory**: grep-audit for `exec`/`spawn` call sites; assert a
   single shared runner/util is used everywhere (`shell: true` and template-built
   command strings are lint-banned — add an ESLint `no-restricted-syntax` rule for
   `child_process` outside `src/util/subprocess.ts` and `src/backend/runner.ts`).
3. **B3 watcher**: symlink-following regression test (symlink cycle in a watched dir);
   1 GB sparse-file guard (metadata-only path taken without reading).
4. **B4 store**: permission audit test walking a fully exercised store (post-E2E
   fixture) asserting 0700/0600 everywhere; session-id validation fuzz (20 hostile
   ids) on every public store API.
5. **B5 redaction**: run the full E2E fixture with secrets planted in *every* field
   type (command, OCR line, diff, window title, note, marker) and assert zero
   plaintext occurrences in `payload.md`, logs, `--json` outputs, and provenance.
6. **B6 consent**: code-path proof (test) that `runBackend` is unreachable without a
   consent record (grep + runtime assertion in `pipeline.ts` that
   `verifyConsent` ran — make the runner require a `consentToken` argument minted
   only by the gate).
7. **B8 writer**: re-run the hostile-path corpus (28) against the *integrated*
   pipeline (fake backend emitting each hostile output) — not just unit level.
8. **B9 config**: config file world-readable warning integration test; `custom.argv`
   notice presence.
9. **B10**: dangerous-pattern corpus review — add ≥ 5 new real-world-shaped vectors
   (from public incident write-ups, synthetic form).
10. **Logs**: grep-audit that no `console.*` bypasses the sanitizing logger
    (lint rule), and daemon.log from the E2E fixture contains zero planted secrets.
11. **Dependency posture**: `npm ls --omit=dev --all` reviewed; every runtime dep
    justified in the audit doc (target: ≤ 10 runtime deps); `npm audit` clean at
    high+.

## Acceptance Criteria

- `docs/research/security-audit-v1.md` committed with all rows `pass` (or linked
  follow-up issues for accepted risks, each with owner sign-off note).
- The two new lint rules active in CI and violated-then-fixed evidence in the PR
  (deliberate scratch violation).
- Consent-token mechanism implemented (req 6) and covered by a test that fails when
  the gate is bypassed.
- All new regression tests green in CI.

## Validation

Full CI + `test:local`; reviewer replays at least items 1, 5, 7 locally.

## Dependencies

All of 03–31 (audits the integrated system); before release issues 36/37.

## Non-goals

External pentest, fuzzing infrastructure (v2), CodeQL/scorecard setup (post-v1),
encrypted store (v2).

## Design References

- DESIGN §14 (entire), §15.1; ADR-004, ADR-007
