# Title

`skillsmith generate` end-to-end orchestration

## Summary

Wire the full generation pipeline behind `skillsmith generate <id>`: timeline → digest
→ redaction → consent → backend → parse → validate → write → provenance, with
`--dry-run`, `--backend`, `--name`, `--out`, `--yes`, `--json`.

## Context

Every stage exists as a module (16–18, 20–30); this issue is composition, flag surface,
failure choreography, and rerun semantics. DESIGN §3 (flags), §10–§12 (pipeline
order) are normative. The pipeline order is security-relevant: consent strictly
precedes any backend spawn.

## Scope

- `src/cli/commands/generate.ts` + `src/synth/pipeline.ts`.

## Detailed Requirements

1. Preconditions: session exists and is `stopped` (else exit 3); backend resolves
   (flag > config; unknown → exit 4); `--out` (or `generate.outputDir`) exists.
2. Stage order (each stage logs a one-line progress note to stderr):
   1. allocate `runId` + provenance skeleton (30);
   2. build digest (16–18) with config budget;
   3. redaction (20) — reuse existing fresh report when `sourceDigestSha256` matches,
      else re-run; **abort on any redaction error** (fail-closed, exit 1);
   4. render preamble (27), assemble payload (21);
   5. `--dry-run` → stop: print payload path + redaction summary; provenance state
      stays `consented`-pending (record `dryRun: true` note in provenance error-free
      finish); exit 0;
   6. consent gate (21) — declined → exit 5, provenance `failed` with
      `error: "consent-declined"`;
   7. run backend (22–26) with `generate.timeoutSeconds`; persist raw output to
      `backend-output.raw` (0600) **before** parsing;
   8. parse sentinel output (28) with `allowScripts` from config;
   9. write skill (28) into `--out`/config dir, honoring `--name`;
   10. validate written dir (29); validation errors → keep the directory, print
       results, exit 7 (draft exists but is non-conformant — user decides);
   11. finalize provenance (`written`, validator verdict, lowConfidence flag,
       `--with-provenance` copy).
3. Failure choreography: any stage failure records provenance `failed` with the stage
   name + sanitized error; partial artifacts stay in `generated/<runId>/` for
   diagnosis; the skill output dir is only created at stage 9's atomic rename (28) —
   no partial skill dirs ever.
4. Rerun semantics: each invocation = new runId; nothing from previous runs is reused
   except the redaction report freshness rule (stage 3).
5. `--json` output (machine): `{runId, state, skillDir?, skillName?, validator?,
   payloadSha256, backend: {id, model}, lowConfidence?}` — exactly this shape on both
   success and validation-failure (state discriminates).
6. Human success output ends with actionable next steps: skill path, `skillsmith
   validate` hint, review-before-install banner (B10), low-confidence warning when
   flagged.

## Acceptance Criteria

- Integration tests (fixture session + custom fake backend from 26): happy path
  produces skill dir + green validation + provenance `written` (deep JSON assertions);
  `--dry-run` sends nothing (fake backend records invocations — assert zero);
  declined consent → exit 5, no backend spawn; backend timeout → exit 6, provenance
  `failed:backend`, raw absent; invalid sentinel output → exit 6 with
  `backend-output.raw` preserved; validator-failing output (fake backend emits bad
  frontmatter) → exit 7, dir exists, provenance records verdict.
- Consent-precedes-spawn proven by invocation-recording fake backend across all paths.
- `--name` override propagates to dir + frontmatter + validation.
- Progress lines appear on stderr only; stdout carries only `--json` payload in json
  mode.

## Validation

`npm test`; full manual run on a real recorded session with a real backend
(`test:local` tier), transcript attached to PR (redacted).

## Dependencies

16, 17, 18, 20, 21, 22, 26 (min for tests), 27, 28, 29, 30; adapters 23–25 for real
use.

## Non-goals

Multi-run comparison UX, regeneration-with-feedback loops (v2), parallel generation.

## Design References

- DESIGN §3, §10, §11, §12; ADR-004, ADR-007
