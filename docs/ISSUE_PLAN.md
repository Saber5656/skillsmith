# skillsmith — v1 Issue Plan

- Status: accepted 2026-07-07
- Design: `docs/DESIGN.md` (normative); decisions in `docs/decisions/`
- Issue drafts: `docs/issues/NN-*.md` (one GitHub Issue per file, English)

## v1 completion statement

When issues **01–37** are all completed and validated, skillsmith v1 is done:
a macOS (14+) CLI installed via `npm install -g skillsmith` that

1. records work sessions (zsh commands, periodic screenshots with OCR text, file
   changes and git snapshots) into a locally stored, permission-hardened session store;
2. distills a stopped session into a deterministic digest, applies a mandatory
   fail-closed redaction pass, and shows the exact payload for explicit user consent;
3. generates a draft skill directory with a spec-conformant `SKILL.md` by delegating to
   a user-chosen agent CLI (`codex` default, `claude`, `gemini`, or custom command)
   without skillsmith ever holding API keys;
4. validates the result against the Agent Skills specification, records full
   provenance (payload hash, consent, backend identity, validator verdict);
5. ships with guided setup (`init`), diagnostics (`doctor`), retention (`sessions
   prune`), a security audit against the documented threat model, CI, E2E tests,
   user documentation, and a provenance-enabled npm release pipeline.

Anything not covered by these issues is explicitly a non-goal (DESIGN §1.5) or a
deferred v2 item (below). Newly discovered implementation unknowns may add issues; the
known candidates are listed at the end.

## Issue list — recommended execution order

| # | File | Title | Wave |
|---|---|---|---|
| 01 | `issues/01-project-scaffolding.md` | Project scaffolding and TypeScript toolchain | 0 |
| 02 | `issues/02-ci-pipeline.md` | CI pipeline (lint, typecheck, test, build, audit) | 0 |
| 06 | `issues/06-cli-skeleton-and-errors.md` | CLI skeleton, error taxonomy, exit codes, logging | 0 |
| 03 | `issues/03-config-loader-and-paths.md` | Path resolution and YAML config loader | 0 |
| 04 | `issues/04-event-model-and-jsonl.md` | Event model and JSONL utilities | 0 |
| 05 | `issues/05-session-store-and-manifest.md` | Session store, manifest state machine, locking | 0 |
| 07 | `issues/07-record-lifecycle-and-daemon.md` | Record lifecycle commands and daemon | 1 |
| 08 | `issues/08-zsh-hook-and-installer.md` | Zsh shell hook and installer | 1 |
| 09 | `issues/09-shell-ingest-normalization.md` | Shell ingest reader and normalization | 1 |
| 10 | `issues/10-screen-capturer.md` | Screen capturer (screenshots, metadata, denylist, TCC) | 1 |
| 11 | `issues/11-ocr-queue-and-engine-interface.md` | OCR engine interface and queue | 1 |
| 12 | `issues/12-vision-jxa-ocr-adapter.md` | Vision OCR adapter via JXA (+ spike gate) | 1 |
| 13 | `issues/13-tesseract-ocr-adapter.md` | Tesseract OCR adapter | 1 |
| 14 | `issues/14-file-activity-watcher.md` | File activity watcher with bounded diffs | 1 |
| 15 | `issues/15-git-snapshots.md` | Git snapshots at start/stop | 1 |
| 16 | `issues/16-timeline-builder-segmentation.md` | Timeline builder and segmentation | 2 |
| 17 | `issues/17-screen-text-distillation.md` | Screen text distillation (OCR dedup) | 2 |
| 18 | `issues/18-digest-generator-budgeting.md` | Digest generator with token budgeting | 2 |
| 19 | `issues/19-redaction-rule-library.md` | Redaction rule library and entropy detector | 2 |
| 20 | `issues/20-redaction-engine-and-command.md` | Redaction engine, report, `redact` command | 2 |
| 21 | `issues/21-consent-gate-and-preview.md` | Consent gate and payload preview | 2 |
| 22 | `issues/22-backend-abstraction-runner.md` | Backend abstraction and hardened runner | 3 |
| 26 | `issues/26-custom-backend-adapter.md` | Custom command backend adapter | 3 |
| 23 | `issues/23-codex-backend-adapter.md` | Codex backend adapter (default) | 3 |
| 24 | `issues/24-claude-backend-adapter.md` | Claude Code backend adapter | 3 |
| 25 | `issues/25-gemini-backend-adapter.md` | Gemini CLI backend adapter | 3 |
| 27 | `issues/27-generation-prompt-template.md` | Generation prompt template | 3 |
| 28 | `issues/28-output-parser-safe-writer.md` | Output parser and safe skill writer | 3 |
| 29 | `issues/29-skill-validator-command.md` | Skill validator and `validate` command | 3 |
| 30 | `issues/30-provenance-records.md` | Provenance records | 3 |
| 31 | `issues/31-generate-command-orchestration.md` | `generate` end-to-end orchestration | 3 |
| 32 | `issues/32-init-and-doctor-commands.md` | `init` and `doctor` commands | 4 |
| 33 | `issues/33-storage-retention-prune.md` | Storage retention: `sessions prune` | 4 |
| 35 | `issues/35-e2e-test-harness.md` | E2E test harness and fixtures | 4 |
| 34 | `issues/34-security-hardening-audit.md` | Security hardening audit | 4 |
| 36 | `issues/36-user-documentation.md` | User documentation set | 4 |
| 37 | `issues/37-release-pipeline.md` | Release pipeline (npm provenance) | 4 |

Ordering notes: 06 before 03 only because 03's CLI wiring plugs into 06's skeleton —
their module layers are independent and they may proceed in parallel. 26 lands before
23–25 so the fake-backend testing path exists for every later issue.

## Dependency table

| Issue | Depends on (hard) | Enables |
|---|---|---|
| 01 | — | all |
| 02 | 01 | all (gates PRs) |
| 03 | 01 | 05, 07, 10, 11, 14, 20, 21, 22, 26, 32, 33 |
| 04 | 01 | 05, 07, 09, 14, 15, 16 |
| 05 | 01, 03, 04 | 07, 08, 20, 21, 30, 32, 33 |
| 06 | 01 | all CLI-surfaced issues (07, 20, 21, 28–33) |
| 07 | 03, 04, 05, 06 | 08, 09, 10, 11, 14, 15, 32, 33 |
| 08 | 05, 07 | 09, 32 |
| 09 | 04, 05, 07, 08 | 16 (data), 35 |
| 10 | 03, 04, 05, 07 | 11, 32, 35 |
| 11 | 03, 04, 07, 10 | 12, 13, 17, 32 |
| 12 | 11 | 13 (fixtures), ADR-006 resolution |
| 13 | 11, 12 | — |
| 14 | 03, 04, 07 | 16 (data), 35 |
| 15 | 04, 07 | 16 (data) |
| 16 | 04, 05 | 17, 18, 20 |
| 17 | 16, 11 | 18 |
| 18 | 16, 17 | 20, 31 |
| 19 | 01 | 20 |
| 20 | 03, 05, 06, 16–19 | 21, 31, 34 |
| 21 | 03, 05, 06, 20 | 31, 34 |
| 22 | 03, 06 | 23–26, 31, 32 |
| 23 | 22 | 31 (real use) |
| 24 | 22 | 31 (real use) |
| 25 | 22 | 31 (real use) |
| 26 | 22 | 31 (tests), 35 |
| 27 | 03 | 21 (preamble), 24/25 (pointer), 28 (sentinels), 31 |
| 28 | 06, 27 | 31 |
| 29 | 06 | 31, 35 |
| 30 | 05, 21, 29 | 31 |
| 31 | 16–18, 20, 21, 22, 26–30 | 32, 34, 35, 36 |
| 32 | 03, 05–08, 10, 11, 22, 26 | 36 |
| 33 | 03, 05, 06, 07 | 36 |
| 34 | 03–31 (integrated system) | 37 (release gate) |
| 35 | 26, 29, 31 | 34, 37 |
| 36 | 31, 32, 33 | 37 |
| 37 | 01, 02, 34, 35, 36 | release |

Soft dependencies (parallelizable with coordination): 12/13 share a fixture set;
23–25 mirror one another's error-mapping pattern; 16/17/18 form a chain but 19 is
independent of all of them.

## Implementation waves

| Wave | Theme | Issues | Exit criterion |
|---|---|---|---|
| 0 | Foundations | 01, 02, 03, 04, 05, 06 | CI green; stub CLI with full `--help`; store CRUD on temp dirs proven |
| 1 | Recording | 07, 08, 09, 10, 11, 12, 13, 14, 15 | A real session on a real mac records shell+screen+OCR+file+git events; crash recovery works; ADR-006 spike resolved |
| 2 | Processing | 16, 17, 18, 19, 20, 21 | `redact` produces a byte-stable redacted digest + report from the fixture session; consent flow demonstrable |
| 3 | Generation | 22, 26, 23, 24, 25, 27, 28, 29, 30, 31 | `generate` end-to-end with fake backend green in CI; with ≥ 1 real backend locally |
| 4 | Hardening & release | 32, 33, 35, 34, 36, 37 | Security audit doc all-pass; E2E green both OSes; docs walkthrough passes; release dry-run green |

Within a wave, issues without mutual dependencies may run in parallel by different
implementation agents; each issue is sized for one focused agent task.

## Coverage table (DESIGN.md → issues)

| DESIGN section | Covered by |
|---|---|
| §1 overview/goals/non-goals | plan-level (this file), 36 (user-facing statement) |
| §2 architecture & module map | structural — every implementation issue references its row |
| §3 CLI surface & exit codes | 06 (skeleton/codes), 07, 20, 21, 29, 31, 32, 33 (commands) |
| §4 configuration | 03 (schema/loader), 32 (scaffold) |
| §5.1–5.2 paths/ids | 03, 05 |
| §5.3–5.4 events | 04 |
| §5.5–5.6 manifest/state machine | 05, 07 |
| §6 daemon | 07 |
| §7.1 shell recorder | 08, 09 |
| §7.2 screen | 10 |
| §7.3 OCR | 11, 12, 13 |
| §7.4 files/git | 14, 15 |
| §8.1 timeline | 16 |
| §8.2 distillation | 17 |
| §8.3 digest/budget | 18 |
| §9 redaction | 19, 20 |
| §10 consent | 21 |
| §11.1 backend contract/runner | 22 |
| §11.2 adapters | 23, 24, 25, 26 |
| §11.3 prompt template | 27 |
| §12.1–12.2 output parsing/writing | 28 |
| §12.3 validation | 29 |
| §12.4 provenance | 30 |
| §13.1 init | 32 |
| §13.2 prune | 33 |
| §13.3 doctor | 32 |
| §14 security model | distributed per-boundary + 34 (cross-cutting audit) |
| §15.1 errors/logging | 06, 20 (sanitizer) |
| §15.2 testing | per-issue + 35 (harness) |
| §15.3 packaging/release | 01, 02, 37 |
| §16 known unknowns | tracked below |
| §17 glossary | n/a (reference) |

Every DESIGN section that specifies buildable behavior maps to at least one issue; no
product behavior lives only in prose outside this plan.

## Validation strategy (whole product)

1. **Per-issue gates**: each issue file carries Acceptance Criteria + Validation; CI
   (02) enforces lint/typecheck/tests on every PR; macOS-only tests are tiered
   (`*.mac.test.ts` on macos runners; `test:local` for TCC-dependent paths).
2. **Determinism spine**: timeline → digest → redaction → payload are pure and
   golden-tested (16–20); payload hashes make any drift visible in provenance.
3. **Security validation**: boundary-local tests land with each issue (hostile-path
   corpus in 28, vector corpus in 19, consent ordering in 21/31); issue 34 re-verifies
   the integrated system against DESIGN §14 with a committed audit document; planted
   secrets in the E2E fixture (35) must never appear in any output artifact.
4. **End-to-end truth**: 35's fixture pipeline runs in CI on every PR from wave 4 on;
   the real-hardware recorder smoke + one real-backend generation is a mandatory
   release-gate step (37's RELEASING checklist), because CI runners cannot grant TCC.
5. **Docs-behavior parity**: 36 requires transcripts from real runs and a scripted
   newcomer walkthrough before release.
6. **Release gate**: 37 verifies pack contents, install smoke, version/changelog
   consistency, and publishes with provenance; merge ≠ release — the first publish is
   an explicit owner decision.

## Deferred v2 items

| Item | Rationale for deferral |
|---|---|
| Linux support (capture adapters, paths) | ADR-002; adapter seams prepared |
| bash / fish shell hooks | ADR-001; zsh is the macOS default |
| PTY wrapper (full terminal output capture) | large correctness surface; value unproven until v1 dogfooding (DESIGN §16.6) |
| Editor extensions (VS Code/JetBrains semantic events) | plugin distribution cost; FS watching covers v1 |
| Screen video capture + VLM image analysis | cost/privacy; OCR path first |
| Direct LLM API adapters (keys) / ollama first-class adapter | ADR-003; `custom` backend is the v1 escape hatch |
| Skill update/merge into existing skill dirs | needs diff/merge UX design |
| Multi-session aggregation into one skill | prompt + provenance design work |
| Interactive review/edit TUI before writing | v1 relies on pager preview + editor |
| `sessions export/import` (portable bundles) | retention covers v1 hygiene |
| Encrypted-at-rest session store | FileVault assumption documented (DESIGN §16.7) |
| Compiled Swift OCR helper | ADR-006 fallback if vision-jxa spike fails or perf demands |
| Homebrew formula, signed tags | post-first-release distribution polish |
| Japanese README / localized docs | after English docs stabilize |
| CodeQL / OpenSSF scorecard / fuzzing | post-v1 security roadmap (34 seeds it) |

## Known unknowns (may spawn new issues during implementation)

| # | Unknown | Owning issue | Contingency |
|---|---|---|---|
| U1 | TCC attribution for `screencapture` inside a detached daemon | 10 (mandatory resolution step) | non-detached capture child fallback, documented |
| U2 | vision-jxa OCR latency/accuracy on real screenshots | 12 (spike gate) | flip default to tesseract per ADR-006; v2 Swift helper issue |
| U3 | Gemini CLI stdin piping + JSON result field | 25 (verification step) | argv-embedding fallback with size guard |
| U4 | `claude --bare` availability on installed versions | 24 (feature-detect) | plain `-p` fallback path |
| U5 | chokidar performance on very large watched trees | 14 (caps) | watchman-style adapter as new issue |
| U6 | Whether command *output* absence degrades skill quality | v1 dogfooding | v2 PTY decision (deferred list) |
| U7 | JXA single-invocation batching limits (arg length, memory) | 11/12 | per-image invocation mode switch |
| U8 | npm trusted publishing friction for first release | 37 | documented owner-side manual steps; classic token as last resort (owner decision) |

## Change control

This plan and `docs/DESIGN.md` are the source of truth. When implementation reality
contradicts an issue: update the issue draft + DESIGN (+ ADR if a decision changes) in
the same PR, then reflect the change in the GitHub Issue (issues are derived
artifacts). New scope requires owner approval before new issues are created.
