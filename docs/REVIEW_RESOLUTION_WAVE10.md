# Wave 10 concrete review-resolution addendum

Repository: Saber5656/skillsmith
Pull request: #1

This addendum is the normative response to the 32 mapped human review findings on
this documentation-only pull request. It gives one implementation path and one
focused verification gate for every finding. The independent review manifest pins
the current head, base, and addendum blob; those mutable identities are intentionally
not repeated here. A head or base change invalidates this addendum and requires a
fresh review.

These are documentation contracts, not claims that product code or tests already
pass. Resolution requires the focused gate, repository full validation, and the
applicable security/privacy handoff to have terminal success for the pinned
identity. No PR review bot is triggered or rerun.

## Mandatory completion gates

The implementation owner must provide evidence for the focused gate and the full
repository validation. Security/privacy review must accept the filesystem,
subprocess, secret, and generated-output boundaries touched by each contract.
Missing, pending, skipped, cancelled, timed-out, stale, or unknown evidence blocks
resolution. The final merge gate separately re-fetches policy, checks, reviews,
head/base, and unresolved threads; this file is not merge authorization.

## Thread contracts

### 1. Honor pause before the shell hook writes commands

- Thread: PRRT_kwDOTN39PM6O7nc8
- Location: docs/issues/08-zsh-hook-and-installer.md:28
- Normative resolution: Each preexec and precmd hook invocation reads the atomically
  published active-session pointer and control state before opening an ingest file.
  When desired is paused, it writes no command event and returns successfully.
- Focused gate: In a zsh fixture, atomically switch control.json to paused between
  two commands and assert that the second command has no pre event while the shell
  exit status is unchanged.

### 2. Wait long enough for graceful daemon shutdown

- Thread: PRRT_kwDOTN39PM6O7ndB
- Location: docs/issues/07-record-lifecycle-and-daemon.md:47
- Normative resolution: record stop waits up to 30 seconds for the daemon's
  stopping-to-stopped drain before sending SIGKILL; after escalation it runs the
  documented crash-recovery finalization exactly once.
- Focused gate: Use a fake daemon that drains at 29 seconds and another that does
  not drain at 30 seconds; assert graceful completion in the first case and one
  SIGKILL plus recovery in the second.

### 3. Mask medium-severity secrets in logs

- Thread: PRRT_kwDOTN39PM6O7ndI
- Location: docs/issues/20-redaction-engine-and-command.md:51
- Normative resolution: The log formatter masks all high- and medium-severity
  secret rules, including openai-key, jwt, and generic-high-entropy, before writing
  daemon or command logs. Raw captured values never enter a log sink.
- Focused gate: Log one vector for each named rule and assert the original value is
  absent while the rule-id replacement is present.

### 4. Validate the hook's parsed session id

- Thread: PRRT_kwDOTN39PM6O7ndM
- Location: docs/issues/08-zsh-hook-and-installer.md:31
- Normative resolution: The hook accepts a session id only when it matches the
  documented lowercase id grammar and contains no slash, dot segment, or NUL.
  Invalid ids are rejected before any path join or file open.
- Focused gate: Run traversal, empty, whitespace, and valid-id fixtures and assert
  that only the valid id reaches the ingest path.

### 5. Special-case delete events before lstat

- Thread: PRRT_kwDOTN39PM6O7ndO
- Location: docs/issues/14-file-activity-watcher.md:33
- Normative resolution: A delete event is handled from the cached prior content
  before any lstat or file read. The watcher records the deletion diff and never
  treats the missing path as a read error.
- Focused gate: Create, cache, delete, and deliver one event; assert one deletion
  diff against the cached content and zero lstat/read calls on the missing target.

### 6. Pass fake-backend controls without stripped env vars

- Thread: PRRT_kwDOTN39PM6O7ndW
- Location: docs/issues/35-e2e-test-harness.md:37
- Normative resolution: The E2E harness selects its fake backend and logging mode
  through explicit test-only argv/configuration. It does not rely on
  SKILLSMITH_* variables, so the production runner's environment allowlist remains
  unchanged.
- Focused gate: Run the local fake-backend test with an empty SKILLSMITH_* process
  environment and assert that the argv/config controls select the fake backend.

### 7. Reconcile bare prune with default retention policy

- Thread: PRRT_kwDOTN39PM6O7nde
- Location: docs/issues/33-storage-retention-prune.md:38
- Normative resolution: A bare prune uses the configured retentionDays value,
  defaulting to 30 days. The explicit older-than flag replaces that age selector,
  and both forms execute the same eligibility, active-session, and path-safety
  checks.
- Focused gate: Run bare prune, configured 10-day prune, and explicit 2-day prune
  against timestamped fixtures and assert the exact selected session ids.

### 8. Spawn daemon from installed CLI path

- Thread: PRRT_kwDOTN39PM6O7ndi
- Location: docs/issues/07-record-lifecycle-and-daemon.md:31
- Normative resolution: The parent resolves the daemon entrypoint to an absolute
  path derived from the installed CLI module URL and passes that path to the child.
  The child is never located by an ambient PATH lookup.
- Focused gate: Launch record start with a hostile PATH and a different cwd and
  assert that the child command contains the installed absolute entrypoint.

### 9. Honor configured shell command byte cap

- Thread: PRRT_kwDOTN39PM6O7ndp
- Location: docs/issues/08-zsh-hook-and-installer.md:35
- Normative resolution: The hook reads the active recording.shell.maxCommandBytes
  value before raw ingest, measures UTF-8 bytes, truncates over-limit commands with
  the fixed truncation marker, and forwards only the bounded payload.
- Focused gate: Use multibyte and ASCII commands at limit and limit+1 and assert
  exact byte length and marker placement for the configured cap.

### 10. Capture diffs for first edits and new files

- Thread: PRRT_kwDOTN39PM6O7ndr
- Location: docs/issues/14-file-activity-watcher.md:39
- Normative resolution: An LRU miss means the prior text is the empty string for a
  text file, so the first edit or newly observed file emits an insertion diff.
  Binary and over-cap files retain metadata-only behavior.
- Focused gate: Deliver a first creation and a first edit after cache eviction and
  assert a unified diff with empty preimage and the full bounded new content.

### 11. Transition through stopping on normal SIGTERM

- Thread: PRRT_kwDOTN39PM6O7ndv
- Location: docs/issues/07-record-lifecycle-and-daemon.md:45
- Normative resolution: SIGTERM atomically changes the manifest to stopping,
  prevents new captures, drains the OCR queue within the bound, writes the final
  snapshot, then changes the manifest to stopped and exits zero. A second signal
  enters the idempotent immediate-flush path.
- Focused gate: Assert the manifest sequence recording -> stopping -> stopped and
  assert that a second SIGTERM produces one final flush and no duplicate snapshot.

### 12. Keep git snapshot truncation schema-valid

- Thread: PRRT_kwDOTN39PM6O7ndz
- Location: docs/issues/15-git-snapshots.md:31
- Normative resolution: Every git snapshot record contains truncated: boolean;
  it is true exactly when output was bounded and false for complete output.
  Consumers use this field instead of inferring truncation from missing text.
- Focused gate: Run below-cap and above-cap snapshot fixtures and validate both
  records against the event schema with the expected boolean.

### 13. Make CI workflow callable before reusing it

- Thread: PRRT_kwDOTN39PM6O7nd4
- Location: docs/issues/37-release-pipeline.md:27
- Normative resolution: The release workflow declares workflow_call in its
  on-section with its required inputs before any other workflow invokes it.
  The callable entry uses the same validation job definitions as a tag release.
- Focused gate: Run the workflow syntax check and a workflow_call dispatch fixture
  and assert that the validation job is created.

### 14. Grant write permission to create GitHub Releases

- Thread: PRRT_kwDOTN39PM6O7neB
- Location: docs/issues/37-release-pipeline.md:37
- Normative resolution: The release job declares contents: write at the narrowest
  workflow/job scope needed to create the GitHub Release; no broader permission is
  added.
- Focused gate: Validate the workflow permission map and assert contents write is
  present on the release job and absent from unrelated jobs.

### 15. Allocate run ids with an exclusive operation

- Thread: PRRT_kwDOTN39PM6O7neI
- Location: docs/issues/30-provenance-records.md:47
- Normative resolution: Run-id allocation creates the candidate directory with an
  exclusive mkdir operation and retries with the next id on EEXIST. It never
  checks existence and then creates non-exclusively.
- Focused gate: Start two allocators concurrently and assert unique directories,
  no overwrite, and contiguous collision retry evidence.

### 16. Add a no-overwrite rule to the file-extraction contract

- Thread: PRRT_kwDOTN39PM6O7st1
- Location: docs/decisions/ADR-007-untrusted-backend-output.md:28
- Normative resolution: The writer normalizes and validates each target path,
  rejects an existing destination before opening it, and never overwrites an
  existing file or follows a destination symlink.
- Focused gate: Supply existing-file, existing-directory, symlink, and traversal
  targets and assert a conflict result with unchanged destination bytes.

### 17. Audit the full dependency tree, not just runtime deps

- Thread: PRRT_kwDOTN39PM6O7suL
- Location: docs/issues/02-ci-pipeline.md:32
- Normative resolution: The CI audit examines the complete lockfile dependency
  tree, including development dependencies, and does not pass an omit-dev flag.
  A high-severity advisory fails the job.
- Focused gate: Run the audit command against a fixture lockfile containing a
  vulnerable dev dependency and assert a failing exit status.

### 18. Use the shared error surface here

- Thread: PRRT_kwDOTN39PM6O7suc
- Location: docs/issues/03-config-loader-and-paths.md:48
- Normative resolution: Config path and parse failures are converted to the
  shared SkillsmithError hierarchy with stable code, message, and cause fields;
  the common CLI handler owns rendering and exit mapping.
- Focused gate: Inject missing, unreadable, and malformed config fixtures and assert
  the same structured error envelope reaches the common handler.

### 19. Make active-session lock write atomic

- Thread: PRRT_kwDOTN39PM6O7suw
- Location: docs/issues/05-session-store-and-manifest.md:42
- Normative resolution: Active-session ownership is acquired by one exclusive
  create operation containing pid, start time, and session id. A live owner causes
  a deterministic conflict; stale recovery requires the documented verification.
- Focused gate: Race two starts and assert one owner, one conflict, and a lock file
  that is never partially written.

### 20. Close the symlink-race in deleteSession

- Thread: PRRT_kwDOTN39PM6O7svC
- Location: docs/issues/05-session-store-and-manifest.md:47
- Normative resolution: deleteSession opens the trusted sessions root and performs
  descriptor-relative, no-follow traversal for every component before removing the
  session. If the platform cannot provide that primitive, it returns
  E_UNSAFE_DELETE and removes nothing.
- Focused gate: Swap a session component to a symlink during deletion and assert
  that the operation fails closed, the outside target is unchanged, and no
  deletion occurs through the link.

### 21. Keep the second-signal path in terminal cleanup

- Thread: PRRT_kwDOTN39PM6O7svL
- Location: docs/issues/07-record-lifecycle-and-daemon.md:45
- Normative resolution: Terminal cleanup is idempotent and shared by normal stop
  and the second-signal path: it flushes bounded state, closes descriptors, marks
  the manifest terminal, and releases the active-session lock exactly once.
- Focused gate: Send SIGTERM twice during OCR drain and assert one cleanup record,
  released lock, terminal manifest, and no leaked child process.

### 22. Target the active zsh rc file, not always ~/.zshrc

- Thread: PRRT_kwDOTN39PM6O7svY
- Location: docs/issues/08-zsh-hook-and-installer.md:53
- Normative resolution: Hook installation edits $ZDOTDIR/.zshrc when ZDOTDIR is
  set and falls back to $HOME/.zshrc when it is unset. It never writes a second
  rc file as a fallback after selecting the target.
- Focused gate: Run installation with a temporary ZDOTDIR and assert only that
  directory's .zshrc receives the marker block; unset ZDOTDIR and assert HOME's
  .zshrc is selected.

### 23. Keep project-dir changes as an unconditional block boundary

- Thread: PRRT_kwDOTN39PM6O7svk
- Location: docs/issues/16-timeline-builder-segmentation.md:34
- Normative resolution: The timeline builder closes the current activity block and
  starts a new one whenever the dominant project directory changes, regardless of
  elapsed time or other event types.
- Focused gate: Feed adjacent activity events in two project directories with a
  sub-five-minute gap and assert two blocks with a boundary at the directory change.

### 24. Preserve shingle-hash dedup stage from DESIGN §8.2

- Thread: PRRT_kwDOTN39PM6O7sv1
- Location: docs/issues/17-screen-text-distillation.md:27
- Normative resolution: Screen distillation normalizes lines, computes consecutive
  shingle hashes, drops hashes already present in the immediately previous capture,
  and only then applies the character/confidence cap.
- Focused gate: Supply two captures with repeated and changed shingles and assert
  that only changed lines survive before cap reduction.

### 25. Keep redaction report schema aligned with DESIGN §9.2

- Thread: PRRT_kwDOTN39PM6O7swC
- Location: docs/issues/20-redaction-engine-and-command.md:45
- Normative resolution: Every redaction report contains generatedAt, sourceEventsHash,
  ruleHits with samplesMasked, allowlistHits, and errors. The source-events hash
  identifies the exact input used for the report.
- Focused gate: Validate a report containing a rule hit, allowlist hit, and engine
  error against the schema and assert all five required fields.

### 26. Keep the shared backend contract aligned with DESIGN §11.1

- Thread: PRRT_kwDOTN39PM6O7swK
- Location: docs/issues/22-backend-abstraction-runner.md:35
- Normative resolution: GenerationBackend.buildInvocation returns argv, stdinFrom,
  resultFrom, and optional resultFile with resultFrom restricted to
  stdout-json-result or file. The shared runner uses execFile semantics, a fresh
  0700 cwd, the exact PATH/HOME/LANG/LC_ALL environment allowlist, process-group
  timeout, 10 MiB stdout/stderr caps, and the documented five error categories.
  The runner does not use stdout-raw or add TERM to the allowlist.
- Focused gate: Run codex/file, claude/stdout-json-result, and custom-config
  fixtures; assert the contract fields, allowlisted environment, result parsing,
  caps, and error taxonomy against DESIGN §11.1.

### 27. Avoid automatic argv fallback for full prompt

- Thread: PRRT_kwDOTN39PM6O7swO
- Location: docs/issues/25-gemini-backend-adapter.md:32
- Normative resolution: Gemini sends the full payload through the supported stdin
  transport by default. It uses an argv prompt only when explicit adapter
  configuration enables it and the payload length passes the platform ARG_MAX
  guard; otherwise the invocation fails closed.
- Focused gate: Run a large-payload fixture with default settings and assert no
  payload bytes occur in argv, then enable the explicit arg mode and assert the
  ARG_MAX guard.

### 28. Keep the refusal path writable

- Thread: PRRT_kwDOTN39PM6O7swT
- Location: docs/issues/27-generation-prompt-template.md:48
- Normative resolution: The writer requires a valid frontmatter name; when the
  backend omits one, it supplies a deterministic valid name
  skill-<first-12-of-payload-sha256> before validation. An explicit --name
  replaces that generated name and is validated by the same rule.
- Focused gate: Parse outputs with a valid name, no name, and invalid name and
  assert the generated hash-based name, explicit override, and validation failure
  respectively.

### 29. Pin temp staging to the target filesystem

- Thread: PRRT_kwDOTN39PM6O7swW
- Location: docs/issues/28-output-parser-safe-writer.md:46
- Normative resolution: Temporary result staging is created beneath the selected
  outputDir with restrictive permissions, and final placement uses a same-parent
  rename. The writer never stages across an unrelated filesystem.
- Focused gate: Set outputDir on a filesystem fixture, inspect the staging parent,
  and assert same-device rename plus cleanup after success and failure.

### 30. Sort prunes by timestamp, not session id

- Thread: PRRT_kwDOTN39PM6O7swY
- Location: docs/issues/33-storage-retention-prune.md:26
- Normative resolution: Prune candidates sort by stoppedAt, then createdAt, then
  session id as a stable tie-breaker. Session id is never the primary age signal.
- Focused gate: Give sessions ids whose lexical order conflicts with timestamps and
  assert oldest timestamp is removed first, with id used only for equal timestamps.

### 31. Keep test:local out of default CI path

- Thread: PRRT_kwDOTN39PM6O7swe
- Location: docs/issues/35-e2e-test-harness.md:58
- Normative resolution: The default CI workflow runs deterministic unit,
  integration, and fake-backend E2E commands; test:local is a separately named
  manual command and is not referenced by the default CI job.
- Focused gate: Parse the workflow and package scripts and assert no default CI
  step invokes test:local while the manual command remains runnable.

### 32. Normalize golden comparison

- Thread: PRRT_kwDOTN39PM6O7swn
- Location: docs/issues/35-e2e-test-harness.md:60
- Normative resolution: Golden comparison passes both actual and expected reports
  through one normalizer that removes runId, generatedAt, createdAt, stoppedAt,
  and other documented timestamp fields, then hashes canonical JSON. It does not
  perform an unrestricted text replacement.
- Focused gate: Compare reports differing only in ids/timestamps and assert equal
  normalized hashes; change a finding detail and assert a mismatch.

## Resolution boundary

This file specifies implementation and verification contracts only. The affected
design and issue text must be updated consistently, and every mapped thread must
receive the owner's evidence-backed reply before it is resolved.
