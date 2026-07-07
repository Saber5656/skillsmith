# Title

Release pipeline: npm publish with provenance

## Summary

Implement the tag-driven release workflow: version/changelog discipline, prepack
verification, npm publish with provenance via OIDC trusted publishing, and the
release checklist.

## Context

DESIGN §15.3 is normative. Supply-chain posture: no long-lived npm tokens in repo
secrets if avoidable (trusted publishing), no postinstall scripts, provenance
attestation on. Per the repository owner's operating rules, **the owner configures the
npm-side trusted-publishing settings manually**; this issue only prepares and
documents the repo side.

## Scope

- `.github/workflows/release.yml`, `docs/RELEASING.md`, package.json release fields,
  SHA-pinning pass over all workflows.

## Detailed Requirements

1. `release.yml`: trigger `push: tags: ['v*']`;
   - job `verify`: full CI suite (reuse via `workflow_call` from 02's workflow) +
     `npm pack` + install-smoke (global install into a temp prefix →
     `skillsmith --version` + `skillsmith validate test/fixtures/skills/valid-minimal`);
   - job `publish` (needs verify, environment `release`):
     `permissions: {contents: read, id-token: write}`;
     `npm publish --provenance --access public`; **no** `NODE_AUTH_TOKEN` if trusted
     publishing is configured — the workflow must fail with a clear message when OIDC
     exchange fails, and `docs/RELEASING.md` explains the owner's one-time npm setup
     steps (do not attempt them from CI);
   - job `github-release`: create a GitHub Release from the tag with the
     `CHANGELOG.md` section body (script extracts the matching version heading).
2. Version discipline: tag must equal `package.json` version (verify job asserts);
   versions follow SemVer starting `0.1.0`; `CHANGELOG.md` must contain the version
   section (asserted).
3. SHA-pin all third-party actions in **all** workflows (02's noted follow-up) with a
   comment carrying the human-readable version.
4. `docs/RELEASING.md` checklist: green CI on main; `npm run test:local` executed on
   real hardware for this commit (recorder smoke + one real-backend generation) with
   evidence linked; CHANGELOG updated; version bump commit; tag + push; post-publish
   verification (`npm view skillsmith version`, provenance badge on npmjs.com);
   rollback procedure (`npm deprecate`, never unpublish after 72 h).
5. Dry-run mechanism: `workflow_dispatch` input `dry_run: true` runs verify + pack +
   `npm publish --dry-run` (no tag needed) for pipeline testing before the first real
   release.

## Acceptance Criteria

- `workflow_dispatch` dry run green end-to-end (link in PR).
- Tag-format mismatch and version/tag disagreement each fail the verify job (scratch
  evidence).
- All workflows SHA-pinned (grep assertion in a small lint script run in CI).
- `docs/RELEASING.md` reviewed by the owner (explicit sign-off comment on the PR) —
  the owner performs the npm trusted-publishing configuration before the first
  release.
- Install-smoke passes on the macOS runner.

## Validation

Dry-run workflow evidence + checklist walkthrough. First real publish happens only
after the owner's manual npm setup and explicit go decision (merge ≠ release).

## Dependencies

01, 02, 35, 36; final issue of v1.

## Non-goals

Homebrew formula (v2), signed git tags enforcement (owner preference), publishing
`@skillsmith/*` sub-packages (single package in v1).

## Design References

- DESIGN §15.3; ADR-002, ADR-008
