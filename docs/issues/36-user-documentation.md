# Title

User documentation set: README, PRIVACY, SECURITY, CONTRIBUTING

## Summary

Write the public-facing documentation for the OSS release: a complete README
(install → quickstart → workflow → configuration), a plain-language PRIVACY.md, a
SECURITY.md with reporting policy, and CONTRIBUTING.md.

## Context

The repo README is currently one line. Release posture (public OSS) makes honest,
security-forward documentation part of the product (DESIGN goals G3/G6, §14). Docs
must match implemented behavior exactly — this issue lands after the pipeline is
stable.

## Scope

- `README.md` (rewrite), `PRIVACY.md`, `SECURITY.md`, `CONTRIBUTING.md`,
  `CHANGELOG.md` scaffold. No website.

## Detailed Requirements

1. `README.md` sections, in order: what it is (2 sentences + the pipeline diagram
   from DESIGN §2 simplified); requirements (macOS 14+, Node 20+, zsh, an agent CLI);
   install (`npm install -g skillsmith`); 5-minute quickstart (init → record →
   work → stop → generate, with real command transcripts); what gets recorded — a
   **prominent table** (source, what, where stored, what can leave the machine);
   privacy model summary (3 bullets: local-only by default, mandatory redaction +
   consent preview, you choose the backend) linking PRIVACY.md; command reference
   table (from DESIGN §3); configuration reference (generated from DESIGN §4 —
   verbatim defaults); troubleshooting (TCC, hook, backend auth — doctor first);
   platform status (macOS only, v2 plans); license badge.
2. `PRIVACY.md` (plain language, no legalese): exactly what each recorder captures
   with examples; storage location and permissions; retention/prune; what is sent
   where and when (only at generate, only redacted digest, only after consent, to the
   CLI you chose — and that CLI's own privacy properties apply); known limitations
   (redaction is best-effort, OCR captures third-party content on screen — meeting
   etiquette note); how to purge everything (`sessions prune`, hook uninstall, rm -rf
   paths listed).
3. `SECURITY.md`: supported versions table; private reporting via GitHub Security
   Advisories (no email in v1); 90-day coordinated disclosure default; scope notes
   (what counts: store perms, redaction bypass, writer path escape, consent bypass);
   link to DESIGN §14 threat model.
4. `CONTRIBUTING.md`: dev setup, test tiers (`test` vs `test:local` and when each is
   required), golden-update procedure (35), security-sensitive-area review rule
   (changes under `src/redact`, `src/consent`, `src/backend/runner`, writer path
   validation require an explicit security note in the PR), conventional commit
   style, DCO not required.
5. `CHANGELOG.md`: Keep-a-Changelog scaffold with `[Unreleased]`.
6. Every command transcript in the docs is generated from a real run (copy-paste,
   then redact usernames) — no invented output.

## Acceptance Criteria

- A newcomer test (scripted): following README quickstart verbatim on a machine with
  prerequisites succeeds through `generate` (documented as a manual validation with
  evidence).
- README "what gets recorded" table reviewed against the implemented event schemas
  (5.4) — field-level accuracy.
- Links valid (`lychee`-style link check run once locally; no CI job in v1).
- Docs mention no unimplemented feature; grep for command names against DESIGN §3.

## Validation

Manual newcomer walkthrough evidence + reviewer read-through.

## Dependencies

31, 32 (behavior stable); before 37.

## Non-goals

Docs site, tutorials/videos, localized docs (English only; Japanese README v2
candidate).

## Design References

- DESIGN §1, §3, §4, §14; PRIVACY content ← ADR-001/004; SECURITY ← DESIGN §14
