# Title

Provenance records for generation runs

## Summary

Implement `src/synth/provenance.ts`: machine-readable `provenance.json` and
human-readable `PROVENANCE.md` for every generation run, linking session data, digest
and payload hashes, consent, backend identity, and validation verdict.

## Context

DESIGN §12.4 is normative; goal G5 (auditability) and boundary B10 (generated skills
are drafts pending human review) depend on this record. Provenance stays in the
session store by default — it may reference sensitive context and must not silently
ship inside shared skill directories.

## Scope

- Provenance writer + reader; `sessions show` extension to list runs. Orchestration
  (31) calls the writer at each pipeline stage.

## Detailed Requirements

1. `provenance.json` (0600, in `generated/<runId>/`), written incrementally
   (state field advances; atomic tmp+rename each write):
   ```json
   {
     "v": 1, "runId": "g-001", "state": "consented|generated|written|failed",
     "sessionId": "ss-…", "sessionTimeRange": {"start": "…", "end": "…"},
     "counts": {"events": 0, "blocks": 0, "shellCommands": 0, "captures": 0, "fileChanges": 0},
     "digestSha256": "…", "payloadSha256": "…",
     "redaction": {"reportPath": "…", "ruleHits": [{"ruleId": "…", "count": 0}]},
     "backend": {"id": "codex", "version": "…", "model": null},
     "promptTemplate": "skill-synthesis-v1",
     "consent": {"mode": "interactive", "consentedAt": "…"},
     "validator": {"valid": true, "errors": 0, "warnings": 3} ,
     "output": {"skillName": "…", "skillDir": "…", "lowConfidence": false},
     "reviewed": false,
     "createdAt": "…", "finishedAt": "…", "error": null
   }
   ```
2. `PROVENANCE.md`: human rendering of the same data — summary table + "this skill was
   machine-generated from a recorded session and has not been reviewed" banner + next
   steps (review checklist referencing the validator warnings).
3. `runId` allocation: `g-` + zero-padded increment scanning existing `generated/`
   entries (no reuse; `declined.json` runs still consume their id).
4. Reader `listRuns(sessionId)` / `readProvenance(sessionId, runId)` with zod
   validation; `sessions show` gains a runs table (runId, state, backend, skillName,
   valid).
5. Optional copy-out: `generate --with-provenance` copies `PROVENANCE.md` (only the
   md, never the json) into the skill directory. Default off (DESIGN §12.4).
6. `reviewed` stays `false` in v1 (no review command yet — v2); field exists so the
   format is stable.

## Acceptance Criteria

- Writer tests: incremental state advance persists correct partial data at each stage
  (consented → generated → written; failed path records `error` string); atomicity
  (tmp file never left behind).
- runId allocation with gaps/declined runs.
- `PROVENANCE.md` golden fixture (stable rendering).
- `sessions show --json` includes the runs array (schema snapshot).
- `--with-provenance` copies only the md; default copies nothing (fs assertions).

## Validation

`npm test`.

## Dependencies

05, 21 (consent record), 29 (validator result shape); orchestrated by 31.

## Non-goals

Review workflow/`reviewed` mutation (v2), provenance signing (v2), embedding metadata
into SKILL.md frontmatter (spec `metadata` field is client-owned; avoided in v1).

## Design References

- DESIGN §12.4, §14.1 B10; ADR-007
