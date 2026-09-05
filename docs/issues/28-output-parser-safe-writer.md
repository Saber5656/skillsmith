# Title

Backend output parser and safe skill writer

## Summary

Implement `src/synth/parser.ts` + `skillWriter.ts`: parse the backend's sentinel-format
output into files, enforce the path/size allowlist, and write the skill directory
atomically with no executable bits.

## Context

This is security boundary B8 and the core of ADR-007: backend output is untrusted
input. DESIGN §12.1–§12.2 are normative.

## Scope

- Parser + writer as pure-ish modules (writer touches fs). Validation (29) runs after;
  orchestration (31) wires them.

## Detailed Requirements

1. Parser `parseSentinelOutput(raw): {files: [{relPath, content}], warnings[]}`:
   - Grammar: a file block starts with a line exactly matching
     `^===SKILLSMITH-FILE: (.+)===$` and ends at the next line exactly
     `^===SKILLSMITH-END===$`. Text outside blocks is ignored but counted
     (`strayChars` warning when > 200 chars — models often add chatter).
   - Unterminated block, nested start, duplicate relPath → `BackendError('parse-failure')`
     (raw preserved by 31 at `backend-output.raw`).
   - `<!-- SKILLSMITH:LOW-CONFIDENCE -->` marker as first body content of SKILL.md →
     `lowConfidence: true` flag in result (warning surfaced by 31).
2. Path validation (before any fs interaction; reject → parse-failure naming the path):
   - must be relative; reject absolute, `..` segments anywhere, backslashes, NUL,
     leading `~`, `.git/` prefix; NFC-normalize then re-check.
   - allowlist: `^SKILL\.md$` or `^references/[a-z0-9][a-z0-9-]*\.md$`; additionally
     `^scripts/[a-z0-9][a-z0-9-]*\.(sh|py)$` **only** when `allowScripts` option true.
   - caps: ≤ 10 files, each ≤ 512 KiB (UTF-8 bytes), total ≤ 2 MiB.
3. Writer `writeSkill({files, outputDir, nameOverride?}): {skillDir, skillName, renamed}`:
   - Determine `skillName`: `nameOverride` or frontmatter `name` of SKILL.md (parse
     YAML frontmatter leniently just for `name`; absence → writer error telling the
     user to rerun or pass `--name`).
   - With `nameOverride`: rewrite the frontmatter `name` to match (string-level,
     preserve rest byte-exact) — directory and frontmatter must agree (spec).
   - Stage into a temp dir first, then a single `rename` into
     `<outputDir>/<skillName>`; target existing → error (exit 1) with `--name`/`--out`
     hint; **no overwrite, no merge** in v1.
   - File modes 0644, directory 0755 (output is meant to be shared, unlike the store);
     never any exec bit (assert after write).
   - `outputDir` must exist and be a directory; resolved path may be anywhere the user
     chooses (it's their explicit flag/config), but the final `skillDir` must be a
     direct child of it (defense against crafted names — `skillName` must also match
     the spec name regex before it touches the fs, even from `--name`).
4. Both modules must be fuzz-friendly pure functions where possible; the path
   validator is exported standalone for reuse and testing.

## Acceptance Criteria

- Parser tests: happy path (2 files + chatter → files + strayChars warning);
  unterminated/nested/duplicate → parse-failure; low-confidence marker detection;
  CRLF-tolerant sentinel matching (`\r\n` input).
- Path validator table-driven test with ≥ 20 hostile fixtures (`../x`, `/etc/passwd`,
  `references/../SKILL.md`, `scripts/a.sh` without allowScripts, `SKILL.md./`, NUL,
  backslash, unicode homoglyph normalization case, 300-char name, `.git/hooks/x.md`) —
  all rejected with the offending path named.
- Writer tests: staging+rename atomicity (partial-write simulation leaves no target
  dir); name override rewrites frontmatter + dir consistently; existing target →
  error; mode assertions (no exec bits); total/size caps.
- Round-trip with 27's golden template output shape.

## Validation

`npm test`.

## Dependencies

06 (errors), 27 (sentinel constants shared from one module — export them from
`promptTemplate.ts` and import here; single source of truth).

## Non-goals

Spec validation beyond `name` (29), provenance (30), merge/update of existing skills
(v2).

## Design References

- DESIGN §12.1, §12.2, §14.1 B8; ADR-007
