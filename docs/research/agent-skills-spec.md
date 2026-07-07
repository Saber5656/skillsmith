# Research: Agent Skills / SKILL.md specification

- Status: verified
- Verified on: 2026-07-07
- Source: <https://agentskills.io/specification> (fetched full text)
- Impact: defines the output contract of skillsmith (`skillsmith generate`, `skillsmith validate`)

## Why this matters

skillsmith's primary deliverable is a skill directory containing a `SKILL.md` file. The
generator prompt, the output parser, and the validator (issues 27–29) must implement the
constraints below exactly. This document is the normative extract; if the upstream spec
changes, update this file first, then the validator.

## Directory structure

A skill is a directory containing, at minimum, a `SKILL.md` file:

```
skill-name/
├── SKILL.md          # Required: metadata + instructions
├── scripts/          # Optional: executable code
├── references/       # Optional: documentation
├── assets/           # Optional: templates, resources
└── ...               # Any additional files or directories
```

## SKILL.md frontmatter (YAML)

| Field | Required | Constraints (verbatim from spec) |
|---|---|---|
| `name` | Yes | Max 64 chars. Lowercase letters, numbers, hyphens only. Must not start/end with hyphen. No consecutive hyphens (`--`). **Must match the parent directory name.** |
| `description` | Yes | 1–1024 chars, non-empty. Describes what the skill does and when to use it. |
| `license` | No | License name or reference to a bundled license file. Keep short. |
| `compatibility` | No | 1–500 chars if present. Environment requirements (product, packages, network). Most skills do not need it. |
| `metadata` | No | Map from string keys to string values. Client-defined extra properties. Use reasonably unique key names. |
| `allowed-tools` | No | Space-separated string of pre-approved tools. **Experimental**; support varies. |

`name` regex used by skillsmith: `^[a-z0-9]+(-[a-z0-9]+)*$` with length 1–64.

## Body content

- Markdown after frontmatter; no format restrictions.
- Recommended sections: step-by-step instructions, input/output examples, edge cases.
- The entire file is loaded when the skill activates → keep it small.

## Optional directories

- `scripts/` — executable code agents can run (Python, Bash, JavaScript common).
- `references/` — on-demand documentation; keep individual files focused.
- `assets/` — templates, images, data files.

## Progressive disclosure (sizing guidance)

1. Metadata (~100 tokens): `name` + `description` loaded at startup for all skills.
2. Instructions (< 5000 tokens recommended): full `SKILL.md` body loaded on activation.
3. Resources: loaded only when required.

Spec recommendation: **keep SKILL.md under 500 lines**; move detail to referenced files.

## File references

Use relative paths from the skill root, e.g. `references/REFERENCE.md`. skillsmith's
validator must check that relative links resolve inside the skill directory (no `..`
escapes).

## Consequences for skillsmith

| Component | Requirement derived from spec |
|---|---|
| Prompt template (issue 27) | Instruct backend to emit valid frontmatter, keyword-rich description, < 500-line body, optional `references/*.md`. |
| Output parser/writer (issue 28) | Enforce path allowlist `SKILL.md`, `references/*.md` (+ `scripts/*` only when explicitly enabled). Directory name = `name`. |
| Validator (issue 29) | Implement every constraint in the frontmatter table above as an error; sizing guidance (500 lines / ~5000 tokens) as warnings; unknown frontmatter keys as warnings. |
