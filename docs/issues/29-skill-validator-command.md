# Title

Skill validator and `skillsmith validate` command

## Summary

Implement `src/skill/validator.ts` and the `validate <path>` command: full Agent
Skills spec conformance checking with errors/warnings, dangerous-pattern lint, JSON
output, and stable exit codes.

## Context

DESIGN §12.3 enumerates the checks; the spec extract is
`docs/research/agent-skills-spec.md` (normative constraints). ADR-007 item 4 adds the
dangerous-pattern lint. The validator runs standalone (user-supplied skills — useful
for skill-repo CI) and post-generation (31).

## Scope

- Validator module + CLI command. Input: a skill directory or a path to a `SKILL.md`
  (directory inferred as its parent).

## Detailed Requirements

1. Errors (any → overall fail, exit 7):
   - E001 SKILL.md missing/unreadable; E002 frontmatter missing/unparsable YAML;
   - E010 `name` missing; E011 name regex/length violation
     (`^[a-z0-9]+(-[a-z0-9]+)*$`, 1–64); E012 name ≠ parent directory name (skipped
     with a warning when validating a bare SKILL.md file whose parent is ambiguous —
     document);
   - E020 `description` missing/empty; E021 description > 1024 chars;
   - E030 `compatibility` present and (empty or > 500 chars);
   - E031 `metadata` present and not a flat string→string map;
   - E032 `allowed-tools` present and not a string;
   - E040 body empty (nothing after frontmatter);
   - E050 relative link target escapes the skill directory (resolve each Markdown
     link/image whose target has no scheme and no leading `#`).
2. Warnings (reported, exit stays 0 unless `--strict`):
   - W101 unknown frontmatter key (anything beyond the six spec fields);
   - W102 SKILL.md > 500 lines; W103 body > 20 000 chars (~5000-token proxy);
   - W110 relative link target missing on disk;
   - W120 dangerous pattern hits, one per match, with rule id and line:
     `dp-curl-pipe-sh` (`(curl|wget)[^\n|]*\|\s*(ba|z)?sh`), `dp-rm-rf`
     (`\brm\s+-[a-z]*rf?[a-z]*\s+(/|~|\$HOME)`), `dp-sudo` (`\bsudo\b`),
     `dp-base64-exec` (`base64\s+(-d|--decode)[^\n]*\|\s*(ba|z)?sh`),
     `dp-cred-read` (`(~|\$HOME)/\.(ssh|aws|config/gh|netrc)\b|\B\.env\b`);
   - W130 `scripts/` present (informational: review executable content), W131 file in
     skill dir with exec bit set.
3. Output:
   - Human: grouped list `error E011 (SKILL.md:2): …` then warnings; summary line
     `N errors, M warnings`.
   - `--json`: `{valid: bool, errors: [{code, message, file, line?}], warnings: […]}`.
   - `--strict` flag: warnings also cause exit 7 (for CI use).
4. Exposed API `validateSkillDir(dir, {strict}): ValidationResult` consumed by 31;
   pure with respect to cwd; never modifies inputs.
5. Fixture corpus `test/fixtures/skills/`: ≥ 12 fixtures — fully valid minimal, valid
   with all optional fields, and one per error class + representative warnings
   (committed as real directories).

## Acceptance Criteria

- Table-driven test running the whole corpus asserting exact code sets per fixture.
- Line numbers correct for frontmatter errors (YAML node positions) and dangerous
  patterns (body offsets) — spot-asserted.
- `validate` on the repo's own generated-fixture skill (from 35 once it lands) is
  green — wire when 35 merges.
- `--json` schema snapshot; exit codes 0 / 7 / 7-with-strict verified via `runCli`.

## Validation

`npm test`; run against 2–3 real-world skills from the user's environment manually and
attach findings (subjective sanity check).

## Dependencies

06; consumed by 31, 35.

## Non-goals

Auto-fixing, style linting beyond the listed rules, YAML schema extensions
(client-specific fields like `metadata` contents are opaque).

## Design References

- DESIGN §12.3, §14.1 B10; ADR-007; docs/research/agent-skills-spec.md
