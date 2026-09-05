# Title

CI pipeline (GitHub Actions): lint, typecheck, test, build, audit

## Summary

Add a GitHub Actions workflow that gates every PR with lint, typecheck, tests on macOS
and Ubuntu, a production build, and a dependency audit.

## Context

skillsmith is macOS-first (ADR-002) but most logic is platform-neutral; running the
pure-logic test subset on Ubuntu keeps portability honest for the v2 Linux port.
Security posture (DESIGN §14, §15.3) requires an audit gate from day one.

## Scope

- `.github/workflows/ci.yml` only. Release workflow is issue 37; E2E wiring is issue 35.

## Detailed Requirements

1. Triggers: `pull_request` and `push` to `main`.
2. Jobs:
   - `check` (ubuntu-latest): `npm ci`, `npm run lint`, `npm run typecheck`,
     `npm run format:check`.
   - `test-macos` (macos-14): `npm ci`, `npm test`, `npm run build`, upload `dist/` as
     artifact (retention 7 days).
   - `test-linux` (ubuntu-latest): `npm ci`, `npm test -- --exclude "**/*.mac.test.ts"`
     — establish the convention now: macOS-only tests are named `*.mac.test.ts`.
   - `audit` (ubuntu-latest): `npm audit --omit=dev --audit-level=high` (fails on
     high/critical). Document in a workflow comment how to handle unfixable advisories
     (temporary `--audit-level=critical` bump requires a linked issue).
3. All jobs: `actions/checkout@v4`, `actions/setup-node@v4` with `node-version: 20` and
   `cache: npm`. Pin actions by major tag; add a comment noting SHA-pinning happens in
   issue 37 (release hardening).
4. `concurrency: { group: ci-${{ github.ref }}, cancel-in-progress: true }`.
5. Workflow permissions: `permissions: { contents: read }` at top level.

## Acceptance Criteria

- CI runs green on a PR containing this workflow.
- A deliberately broken lint (scratch commit, then reverted) fails the `check` job —
  include the red run link in the PR as evidence.
- Fork PRs get no secrets (default behavior; no `pull_request_target` used).

## Validation

Open the PR, observe all four jobs pass; attach links to one green run and the
deliberate red run.

## Dependencies

01.

## Non-goals

Release/publish workflow (37), E2E jobs (35), CodeQL/scorecard (post-v1 hardening).

## Design References

- DESIGN §15.2, §15.3; ADR-002
