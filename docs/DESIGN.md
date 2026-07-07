# skillsmith — v1 Design

Generate SKILL.md files from recorded work sessions across screen, shell, and editor
activity.

- Status: v1 design, accepted 2026-07-07
- Canonical source of truth: this file + `docs/decisions/*.md` + `docs/issues/*.md`
- Issue plan: `docs/ISSUE_PLAN.md`
- Research evidence: `docs/research/*.md`

---

## 1. Product overview

### 1.1 What it does

skillsmith is a macOS CLI that:

1. **Records** a work session: shell commands (zsh hook), periodic screenshots with OCR
   text (Apple Vision), and file changes in declared project directories (watcher +
   git snapshots).
2. **Distills** the session into a redacted, time-ordered digest.
3. **Generates** a draft skill — a directory with a spec-conformant `SKILL.md`
   (Agent Skills format, see `docs/research/agent-skills-spec.md`) — by delegating to a
   locally installed agent CLI (`codex`, `claude`, `gemini`, or custom).
4. **Validates** the result against the Agent Skills specification and records
   provenance (what data, which backend, which consent).

### 1.2 Why

Engineers repeatedly perform procedures that would make excellent reusable agent skills,
but writing SKILL.md files by hand is tedious and lossy. The best source of a skill is a
real, successful execution of the procedure. skillsmith turns "I just did it" into "my
agents can do it now".

### 1.3 Personas and core journeys

| Persona | Journey |
|---|---|
| Agent power user | `skillsmith record start` → does a gnarly one-off procedure (deploy, migration, debugging ritual) → `record stop` → `generate` → reviews draft → installs into `~/.claude/skills` or Codex skills dir |
| Team enabler | Records a golden-path workflow once, generates a skill, reviews/edits, commits it to the team's skills repo |
| Privacy-conscious engineer | Uses denylist + pause/resume during sensitive moments, inspects the exact payload at the consent gate, prunes sessions afterwards |

### 1.4 v1 goals

- G1: One-command session recording with graceful lifecycle (start/stop/pause/resume/mark).
- G2: Three capture sources per ADR-001 with bounded resource usage.
- G3: Mandatory redaction + explicit consent before any data leaves the machine (ADR-004).
- G4: Spec-valid SKILL.md drafts via pluggable agent-CLI backends (ADR-003).
- G5: Full auditability: every generated artifact traces to a payload hash + consent record.
- G6: OSS-release quality: docs, CI, tests, security model, npm publish pipeline.

### 1.5 v1 non-goals

- Linux/Windows support (adapter seams prepared; implementation v2).
- bash/fish shell hooks, PTY/full-terminal-output capture.
- Editor plugins, keystroke logging, input monitoring.
- Screen video recording or VLM (image-to-LLM) analysis; v1 screen understanding is OCR.
- Direct LLM provider APIs (keys) — agent CLIs only.
- Skill *update/merge* into existing skills (v1 always emits a new draft directory).
- Multi-session aggregation into one skill.
- Encrypted-at-rest session store (OS FileVault assumed; see §16 known unknowns).
- Any telemetry, update checks, or network calls by skillsmith itself.

---

## 2. System architecture

```
                       ┌────────────────────────── skillsmith CLI ──────────────────────────┐
                       │  init | doctor | config | record | mark | sessions | redact |      │
                       │  generate | validate                                               │
                       └───────┬──────────────────────────────┬─────────────────────────────┘
                               │ spawns/controls              │ reads
                               ▼                              ▼
   zsh hook ──append──▶ ┌─────────────┐   normalize   ┌──────────────────┐
   (ingest/*.jsonl)     │  recording  │ ────────────▶ │  session store   │
   screencapture ──────▶│   daemon    │               │  (JSONL + files) │
   (screens/*.png)      │  + OCR queue│               └───────┬──────────┘
   fs watcher ─────────▶└─────────────┘                       │
                                                              ▼
                                              ┌────────────────────────────┐
                                              │ timeline → digest builder  │
                                              └───────┬────────────────────┘
                                                      ▼
                                              ┌────────────────────────────┐
                                              │ redaction engine (forced)  │──▶ redaction report
                                              └───────┬────────────────────┘
                                                      ▼
                                              ┌────────────────────────────┐
                                              │ consent gate (preview+yes) │──▶ consent record
                                              └───────┬────────────────────┘
                                                      ▼ (only redacted payload)
                                              ┌────────────────────────────┐
                                              │ backend adapter (subprocess│
                                              │ codex/claude/gemini/custom)│
                                              └───────┬────────────────────┘
                                                      ▼ (untrusted output)
                                              ┌────────────────────────────┐
                                              │ sentinel parser → safe     │
                                              │ writer → skill validator   │──▶ skill dir + provenance
                                              └────────────────────────────┘
```

Module map (repository layout to be created by the issues):

| Path | Responsibility | Issues |
|---|---|---|
| `src/cli/` | command registry, flags, exit codes, output formatting | 06, 07, 20, 21, 29, 31, 32, 33 |
| `src/config/` | path resolution, YAML config schema/loader | 03 |
| `src/store/` | session dirs, manifest state machine, JSONL utils, locks, prune | 04, 05, 33 |
| `src/record/daemon/` | daemon lifecycle, heartbeat, signal handling | 07 |
| `src/record/shell/` | zsh hook (shipped asset), installer, ingest reader | 08, 09 |
| `src/record/screen/` | capture loop, frontmost metadata, denylist, downscale | 10 |
| `src/record/ocr/` | engine interface, queue, vision-jxa + tesseract adapters | 11, 12, 13 |
| `src/record/files/` | fs watcher, diffs, git snapshots | 14, 15 |
| `src/timeline/` | merge, segmentation, screen-text distillation, digest | 16, 17, 18 |
| `src/redact/` | rule library, entropy, engine, report | 19, 20 |
| `src/consent/` | payload preview, gate, consent records | 21 |
| `src/backend/` | backend interface, registry, subprocess runner, adapters | 22–26 |
| `src/synth/` | prompt template, payload assembly, output parser, skill writer, provenance | 27, 28, 30, 31 |
| `src/skill/` | Agent Skills validator/linter | 29 |
| `src/util/` | subprocess, safe-fs, logger, errors | 06, 22, 34 |

---

## 3. CLI surface

Binary: `skillsmith`. All commands support `--json` (machine output to stdout, logs to
stderr), `--verbose`, `--quiet`.

| Command | Purpose |
|---|---|
| `skillsmith init [--no-input]` | guided setup: config scaffold, zsh hook install (confirmed), Screen Recording permission walkthrough, backend detection |
| `skillsmith doctor [--json]` | environment diagnosis (see §13.3) |
| `skillsmith config show\|path` | print effective config / config file path |
| `skillsmith record start [--project <dir>]... [--note <text>]` | start recording session (daemon) |
| `skillsmith record stop` | stop active session |
| `skillsmith record status [--json]` | active session state, counters, recorder health |
| `skillsmith record pause` / `resume` | suspend/resume all capture (privacy pause) |
| `skillsmith mark <label>` | insert a user marker event into the active session |
| `skillsmith sessions list [--json]` | list sessions with state/size/counters |
| `skillsmith sessions show <id> [--json]` | session detail: manifest, block summary |
| `skillsmith sessions delete <id> [--force]` | delete one session directory |
| `skillsmith sessions prune [--older-than <days>] [--max-total-size <MB>] [--dry-run]` | retention cleanup |
| `skillsmith redact <id> [--dry-run] [--json]` | run/refresh redaction, print report |
| `skillsmith generate <id> [--backend <id>] [--name <skill-name>] [--out <dir>] [--dry-run] [--yes] [--json]` | full pipeline: digest → redact → consent → backend → parse → validate → write |
| `skillsmith validate <path> [--json]` | validate a skill directory or SKILL.md file |

Notes:

- `--project` may repeat; defaults to the cwd at `record start`.
- `generate --dry-run` stops after writing the payload + report (no consent, no send).
- `generate --name` overrides the skill name proposed by the backend (writer renames
  directory + frontmatter consistently, then re-validates).
- `record start` when a session is already active → exit 3 with hint.

### 3.1 Exit codes (stable contract)

| Code | Meaning |
|---|---|
| 0 | success |
| 1 | unexpected internal error |
| 2 | usage error (bad flags/args; commander) |
| 3 | session state error (no active session, not found, wrong state) |
| 4 | missing permission/dependency (TCC screen recording, hook not installed, backend missing) |
| 5 | consent declined / non-interactive consent not permitted |
| 6 | backend failure (spawn error, timeout, non-zero exit, unparsable output) |
| 7 | validation failure (generated or user-supplied skill has spec errors) |

---

## 4. Configuration

File: `$XDG_CONFIG_HOME/skillsmith/config.yaml` (default `~/.config/skillsmith/config.yaml`),
mode `0600`. Parsed with a strict schema (zod); unknown keys are errors with a
did-you-mean hint. Every field has a default; an absent file is valid.

```yaml
version: 1                     # config schema version (int, required if file exists)
recording:
  screenshot:
    enabled: true
    intervalSeconds: 10        # 5..300
    maxWidthPx: 1600           # downscale via sips; 800..4000
    appDenylist:               # bundle ids; capture skipped while frontmost
      - com.1password.1password
      - com.apple.keychainaccess
      - com.apple.Passwords
    captureAllDisplays: true   # false → main display only
  shell:
    enabled: true
    maxCommandBytes: 8192
  files:
    enabled: true
    ignore: []                 # merged over built-in defaults (node_modules, .git, dist, …)
    maxFileSizeBytes: 1048576  # files larger than this: metadata-only events
    maxDiffBytes: 65536        # unified diff truncation cap
  limits:
    maxSessionHours: 8         # daemon auto-stops the session
    maxSessionSizeMB: 2048     # capture pauses with a warning event when exceeded
ocr:
  engine: auto                 # auto | vision-jxa | tesseract | off
  languages: [en, ja]
redaction:
  customPatterns: []           # [{id, regex, description?}] — id: ^[a-z0-9-]{1,32}$
  allowlist: []                # exact strings never treated as secrets (e.g. known fake keys)
  redactPaths: true            # /Users/<name> → /Users/USER in all payload text
consent:
  allowNonInteractive: false   # must be true for generate --yes to work
backend:
  default: codex               # codex | claude | gemini | custom
  codex:   { model: null }     # null → CLI's own default
  claude:  { model: null }
  gemini:  { model: null }
  custom:  { argv: [], promptVia: stdin }  # argv: full command array; promptVia: stdin|arg
generate:
  outputDir: "."               # skill dir written to <outputDir>/<skill-name>/
  maxPayloadChars: 240000      # ≈60k tokens; digest builder must fit under this
  allowScripts: false          # ADR-007
  timeoutSeconds: 300
storage:
  retentionDays: 30            # prune eligibility (sessions prune)
```

Precedence: CLI flags > environment (`SKILLSMITH_CONFIG` path override only) > file >
defaults. No other environment-variable configuration in v1.

---

## 5. Data model

### 5.1 Paths

```
$XDG_DATA_HOME/skillsmith/                     (default ~/.local/share/skillsmith) mode 0700
├── active-session.json        # {sessionId, pid, startedAt} + advisory lock file
├── active-session.lock
└── sessions/<session-id>/     # 0700
    ├── manifest.json          # 0600, atomic writes (tmp+rename)
    ├── events.jsonl           # normalized log, daemon is sole writer
    ├── ingest/                # raw shell-hook drops: shell-<tty>-<pid>.jsonl
    ├── screens/000001.png …   # zero-padded capture seq
    ├── ocr/000001.json …      # per-capture OCR result
    ├── diffs/000001.patch …   # bounded unified diffs
    ├── redacted/              # digest.redacted.md, report.json (derived, repeatable)
    ├── generated/<runId>/     # runId = g-001, g-002, …
    │   ├── payload.md         # exact consented payload
    │   ├── consent.json
    │   ├── backend-output.raw
    │   ├── result/<skill-name>/…   # parsed files before copy-out
    │   └── provenance.json / PROVENANCE.md
    └── daemon.log
```

### 5.2 Session id

`ss-<YYYYMMDD>-<HHmmss>-<4 lowercase hex>` (local time at creation), e.g.
`ss-20260707-153012-9f3a`. Regex: `^ss-\d{8}-\d{6}-[0-9a-f]{4}$`.

### 5.3 Event envelope (events.jsonl)

Every line: `{"v":1,"ts":"<ISO-8601 UTC ms>","seq":<int>,"source":"<s>","kind":"<k>","data":{…}}`
`seq` is a strictly increasing integer assigned by the daemon. Readers skip
unparsable lines and report a `corruptLines` count.

### 5.4 Event kinds (normative)

| source | kind | data fields |
|---|---|---|
| session | session.start | `sessionId, note?, projectDirs[], host{os,osVersion,arch}, appVersion` |
| session | session.stop | `reason: "user"\|"limit-hours"\|"crash-recovered"\|"error"` |
| session | session.pause / session.resume | `{}` |
| session | session.marker | `label` (≤ 512 chars) |
| session | session.warning | `code, message` (e.g. `size-limit`, `recorder-error`) |
| shell | shell.command | `shell:"zsh", tty, cwd, command, startedAt, endedAt?, exitCode?, durationMs?, truncated?:bool` |
| screen | screen.capture | `file, displayId, frontmostApp{name,bundleId}, windowTitle?` |
| screen | screen.skipped | `reason:"denylist"\|"paused"\|"error", frontmostApp?{…}, message?` |
| screen | screen.ocr | `captureRef, file, engine, status:"ok"\|"failed", textLength, durationMs` |
| files | file.change | `path, changeType:"create"\|"modify"\|"delete", sizeBytes?, isBinary, hash?("sha256:…"), diffFile?, truncated?:bool` |
| files | git.snapshot | `phase:"start"\|"stop", repoRoot, branch, headSha, dirtyFiles[{path,status}], diffStat{files,insertions,deletions}` |

All schemas are zod-defined in `src/store/events.ts`; the JSON above is normative for
field names and enums.

### 5.5 Manifest (manifest.json)

```json
{
  "v": 1,
  "id": "ss-20260707-153012-9f3a",
  "state": "recording",
  "note": "deploy hotfix to staging",
  "projectDirs": ["/Users/USER/dev/acme"],
  "createdAt": "…", "startedAt": "…", "stoppedAt": null, "stopReason": null,
  "daemon": { "pid": 4242, "startedAt": "…", "heartbeatAt": "…" },
  "recorders": { "shell": "active", "screen": "active", "files": "active" },
  "counters": { "shellCommands": 0, "screenshots": 0, "ocrDone": 0, "fileChanges": 0, "warnings": 0 },
  "appVersion": "0.1.0"
}
```

`recorders.*` ∈ `active | disabled | error`. Counters are updated on heartbeat writes
(every 5 s), not per event.

### 5.6 Session lifecycle state machine

| From | Event | To | Side effects |
|---|---|---|---|
| (none) | `record start` | created | session dir + manifest created, lock acquired |
| created | daemon ready (first heartbeat) | recording | `session.start` event |
| recording | `record pause` | paused | `session.pause` event; capture loops idle |
| paused | `record resume` | recording | `session.resume` event |
| recording/paused | `record stop` | stopping | SIGTERM to daemon |
| stopping | daemon flushed & exited | stopped | `session.stop{reason:user}`, git stop-snapshot, lock released |
| recording/paused | daemon hits `maxSessionHours` | stopping → stopped | `session.stop{reason:limit-hours}` |
| recording/paused | heartbeat stale > 60 s (observed by any CLI command) | stopped | recovery: finalize manifest, `session.stop{reason:crash-recovered}`, lock released |
| created | daemon fails to start (10 s) | failed | error surfaced, lock released |

Illegal transitions are rejected with exit 3. Only `stopped` sessions can be redacted,
generated, pruned. `failed` sessions can only be deleted/inspected.

---

## 6. Recording daemon

- `record start` validates preconditions, creates the session, then spawns
  `skillsmith __daemon <sessionId>` detached (`setsid`-equivalent; stdio →
  `daemon.log`), waits ≤ 10 s for the first heartbeat, prints status, exits.
- The daemon owns: shell-ingest tailer, screenshot loop, OCR queue, fs watcher, git
  snapshots, heartbeat (5 s, atomic manifest write), size/time limit enforcement.
- Signals: SIGTERM → graceful stop (stop capture → drain OCR queue ≤ 30 s → final git
  snapshot → `session.stop` → manifest `stopped` → exit 0). SIGINT same. Second signal
  → immediate flush + exit.
- `record stop` sends SIGTERM, polls manifest ≤ 15 s, escalates SIGKILL then performs
  crash-recovery finalization.
- Pause/resume via control file `sessions/<id>/control.json` `{desired: "recording"|"paused"}`
  written atomically by the CLI and polled by the daemon each second (no IPC socket in v1).
- The single-active-session lock is `active-session.lock` via `O_EXCL` create; stale
  locks (no live pid) are reclaimed by `doctor` or `record start` after verification.

---

## 7. Recorders

### 7.1 Shell (zsh hook)

- Shipped asset `assets/skillsmith-hook.zsh`, installed by `init` into `~/.zshrc`
  between markers `# >>> skillsmith >>>` / `# <<< skillsmith <<<` (idempotent,
  removable by `init --uninstall-hook`).
- Hook behavior (pure zsh, zero subprocess on the hot path, `zsh/datetime` for time):
  - On each `preexec`: read pointer file `~/.local/share/skillsmith/active-session.json`
    (skip everything if absent, `SKILLSMITH_DISABLE=1`, or state ≠ recording);
    append `{"e":"pre","id":"<cmdId>","ts":<epoch.ms>,"cwd":"<pwd>","cmd":"<line>"}` to
    `sessions/<id>/ingest/shell-<tty>-<pid>.jsonl`.
  - On `precmd`: append `{"e":"post","id":"<cmdId>","ts":…,"code":$?}`.
  - Commands > `maxCommandBytes` are truncated with a marker; every write is
    `2>/dev/null || true` — the hook must never break a user's shell.
- Daemon tails `ingest/*.jsonl`, pairs pre/post by `cmdId`, emits `shell.command`
  events; unpaired `pre` after 10 min → event with `exitCode: null`.

### 7.2 Screen

- Loop every `intervalSeconds`: read frontmost app (JXA `NSWorkspace`); if bundle id in
  denylist → emit `screen.skipped{reason:denylist}` and skip; else run
  `screencapture -x -t png [-D n | all displays]` to `screens/<seq>.png`, downscale with
  `sips --resampleWidth <maxWidthPx>`, emit `screen.capture`, enqueue OCR.
- Window title best-effort via CGWindowList (JXA); absent on failure.
- TCC failure detection: capture returns non-zero or produces an implausible image →
  `screen.skipped{reason:error}` + one-time `session.warning`; doctor explains the fix.

### 7.3 OCR queue

- Bounded in-memory queue (max 32 pending; overflow drops oldest with a warning event).
- Worker batches up to 4 images per engine invocation; per-invocation timeout 30 s.
- Results: `ocr/<seq>.json` `{engine, lines:[{text, confidence?, bbox?}], durationMs}`;
  `screen.ocr` event links capture → text.
- Engines per ADR-006: `vision-jxa` (default), `tesseract` (fallback), `off`.

### 7.4 Files

- chokidar watcher per project dir; ignore set = built-in defaults ∪ config ∪ patterns
  from the project's `.gitignore` (best-effort parse).
- On change: hash content (sha256), dedup identical rewrites; for text files ≤
  `maxFileSizeBytes`, produce a unified diff against the previously captured content
  (cache of last content per path, LRU 512 entries) into `diffs/`, truncated at
  `maxDiffBytes`.
- Binary detection: null byte in first 8 KiB → metadata-only event.
- Debounce 500 ms per path; global backpressure: > 50 events/s → coalesce per path,
  emit `session.warning{code:file-storm}` once.
- Git snapshots at start/stop per repo detected in project dirs (`git.snapshot` events).

---

## 8. Timeline and digest

### 8.1 Timeline builder

Reads `events.jsonl` → validated, time-ordered timeline. Segmentation into **activity
blocks**: boundary when gap between consecutive user-activity events (shell.command,
file.change, marker) exceeds 5 min, or when the dominant project dir changes. Each block
gets: time range, dominant app(s), shell command list, file-change summary, distilled
screen text, markers.

### 8.2 Screen text distillation

Consecutive OCR results are near-duplicates. Distillation: normalize lines →
shingle-hash consecutive captures → drop lines already seen in the previous capture
(delta-only) → cap retained screen text per block (default 4000 chars, drop lowest
`confidence` lines first). Window-title timeline is kept separately (compact).

### 8.3 Digest format (input to the LLM)

Deterministic Markdown, sections fixed:

```
# Session digest
meta: session id, date, duration, note, project dirs, counters
## Block 1 — 15:30–15:42 — <dominant app / project>
### Markers        (user-inserted labels)
### Shell          (fenced list: cwd $ command → exit code, duration)
### File changes   (path, change type, +N/−M, truncated diffs for the most-touched files)
### Screen         (delta OCR text, window titles)
## Block 2 — …
```

Budgeting: target ≤ `generate.maxPayloadChars` after redaction. Deterministic reduction
order when over budget: (1) drop screen text of blocks with shell/file activity,
(2) truncate diffs to headers, (3) drop per-block screen text entirely, (4) summarize
oldest blocks to counters-only. Reductions are recorded in the digest header
(`truncation: [...]`) so the LLM and the human know what is missing.

---

## 9. Redaction engine

### 9.1 Built-in rule library (issue 19; ids are stable API)

`aws-access-key-id, aws-secret-key, github-token, gitlab-pat, slack-token,
stripe-key, openai-key, anthropic-key, google-api-key, jwt, pem-block,
ssh-private-key, authorization-header, url-credentials, env-secret-assignment,
generic-high-entropy, email, ipv4-private?` — each rule: `{id, description, regex,
severity: high|medium|low, testVectors: {match[], noMatch[]}}`. `generic-high-entropy`
uses Shannon entropy > 4.0, length ≥ 20, mixed-charset heuristics on token-shaped
strings.

### 9.2 Application

- Fields redacted: shell commands, OCR lines, diffs, window titles, file paths
  (username normalization when `redactPaths`), note, marker labels — i.e. **every**
  string that can enter a payload.
- Replacement: `[REDACTED:<rule-id>:<n>]` with per-session stable numbering (same
  secret → same n) so cross-references survive.
- Config `customPatterns` extend, `allowlist` exempts exact strings.
- Output: `redacted/digest.redacted.md` + `redacted/report.json`
  `{generatedAt, sourceEventsHash, ruleHits: [{ruleId, count, samplesMasked}], allowlistHits, errors: []}`.
- **Fail-closed**: any rule engine error aborts redaction; `generate` refuses to
  proceed without a fresh successful report matching the current digest hash.

---

## 10. Consent gate

1. `generate` assembles digest → runs redaction → writes `generated/<runId>/payload.md`
   (redacted digest + generation instructions preamble — the *exact* stdin bytes for
   the backend).
2. Prints summary table: payload size, block/event counts, redaction hits by rule,
   backend id + model, then opens payload in `$PAGER` (default `less`).
3. Interactive prompt: `Send this payload to backend "codex"? [y/N]` — anything but
   explicit `y`/`yes` → exit 5, run directory kept for inspection.
4. Consent record `consent.json`: `{payloadSha256, backend, model, consentedAt,
   appVersion, mode: "interactive"|"non-interactive"}`.
5. `--yes` allowed only when `consent.allowNonInteractive: true` (ADR-004); recorded as
   `non-interactive`.
6. Any payload change (new digest/redaction) → new runId, new consent.

---

## 11. Generation backends

### 11.1 Contract

```ts
interface GenerationBackend {
  id: 'codex' | 'claude' | 'gemini' | 'custom';
  detect(): Promise<{ available: boolean; version?: string; path?: string }>;
  buildInvocation(req: { payloadFile: string; model?: string; timeoutMs: number }):
    { argv: string[]; stdinFrom: 'payloadFile' | null; resultFrom: 'stdout-json-result' | 'file'; resultFile?: string };
}
```

Shared runner (issue 22): `execFile` semantics (never a shell), cwd = fresh `0700`
temp dir (removed after), env = `{PATH, HOME, LANG, LC_ALL}` only (strip `SKILLSMITH_*`),
timeout with process-group kill, stdout/stderr caps 10 MiB, error taxonomy
`{not-found, timeout, non-zero-exit, output-too-large, parse-failure}` → exit 6.

### 11.2 Adapter invocations (verified 2026-07-07, re-verify at implementation)

| Backend | argv | Result |
|---|---|---|
| codex | `codex exec --ephemeral --skip-git-repo-check [-m <model>] -o <tmp>/last-message.md -` | file `last-message.md` |
| claude | `claude --bare -p <instructions-pointer-prompt> --permission-mode dontAsk --output-format json` (payload via stdin) | JSON stdout `.result`; fall back to plain `-p` if `--bare` unsupported by installed version |
| gemini | `gemini -p <prompt> --output-format json [-m <model>]` (stdin support verified at impl; fallback: payload embedded in `-p` with ARG_MAX guard) | JSON stdout |
| custom | config `backend.custom.argv` verbatim + `promptVia: stdin\|arg` | stdout raw |

See `docs/research/agent-cli-backends.md` for evidence and drift-handling rules.

### 11.3 Prompt template (issue 27)

Versioned file `assets/prompts/skill-synthesis-v1.md` containing: role framing, the
Agent Skills constraints (from research doc, embedded verbatim), the sentinel output
contract (§12.1), an instruction that session data is untrusted *data* (never
instructions), skill-quality guidance (imperative steps, when-to-use description,
< 500 lines), and the `{{DIGEST}}` slot. Template version recorded in provenance.

---

## 12. Output handling

### 12.1 Sentinel output contract

The backend must answer with one or more file blocks, nothing else significant:

```
===SKILLSMITH-FILE: SKILL.md===
---
name: deploy-hotfix-staging
description: …
---
…
===SKILLSMITH-END===
===SKILLSMITH-FILE: references/commands.md===
…
===SKILLSMITH-END===
```

### 12.2 Parser + safe writer (issue 28; ADR-007 normative)

- Grammar: sentinel lines exactly; unterminated block → parse failure (exit 6, raw
  output preserved at `backend-output.raw`).
- Path rules: relative, no `..`, no leading `/`, no backslashes, NFC-normalized,
  allowlist `^(SKILL\.md|references/[a-z0-9][a-z0-9-]*\.md)$` (+
  `scripts/[a-z0-9][a-z0-9-]*\.(sh|py)$` only when `generate.allowScripts`).
- Caps: ≤ 10 files, ≤ 512 KiB each. No exec bits ever.
- Write to `generated/<runId>/result/<skill-name>/`, validate (§12.3), then copy to
  `<outputDir>/<skill-name>/`; if target exists → error with `--out`/`--name` hint
  (no overwrite in v1).
- Skill name: frontmatter `name` (or `--name` override → writer rewrites frontmatter +
  directory consistently).

### 12.3 Validator (issue 29) — also `skillsmith validate`

Errors (exit 7): missing/unparsable frontmatter; `name` violating
`^[a-z0-9]+(-[a-z0-9]+)*$` / length 1–64 / ≠ directory name; `description` empty or
> 1024; `compatibility` > 500; `metadata` not a string→string map; relative links
escaping the skill dir; empty body.
Warnings: unknown frontmatter keys; SKILL.md > 500 lines; body > ~5000-token estimate;
missing link targets; dangerous patterns in content (`curl|wget … | sh`, `rm -rf`,
`sudo`, `base64 -d | sh`-style, reads of `~/.ssh|~/.aws|.env`) — each with rule id.
`--json`: `{errors: [{code, message, loc?}], warnings: [...]}`.

### 12.4 Provenance (issue 30)

`provenance.json` + human `PROVENANCE.md` per run: session id, time range, event/block
counts, redaction rule hits, digest + payload SHA-256, backend `{id, version, model}`,
prompt template version, consent mode + timestamp, validator verdict, `reviewed: false`.
Stored in the session store only (not copied into the skill dir by default).

---

## 13. Supporting commands

### 13.1 init

Interactive: write config scaffold if absent → offer zsh hook install (show the exact
block, require confirmation) → screen-permission walkthrough (test capture, on failure
`open "x-apple.systempreferences:com.apple.preference.security?Privacy_ScreenCapture"`)
→ backend detection table → next-steps summary. `--no-input`: config scaffold +
detection only, no shell/system modification.

### 13.2 storage prune

Eligibility: `stopped`/`failed` sessions older than `retentionDays` (or `--older-than`),
and/or oldest-first until total ≤ `--max-total-size`. Never touches the active session.
Deletion is `rm -rf` of the session directory only after verifying the resolved path is
strictly inside the sessions root and is not a symlink (lstat).

### 13.3 doctor checks

| Check | Failure guidance |
|---|---|
| config parses; version supported | show path + error location |
| data dir perms 0700 (and children) | offer `--fix` chmod |
| zsh hook installed + current version | re-run init |
| screen capture usable (test capture heuristic) | TCC deep link |
| OCR engine resolves (vision-jxa probe / tesseract on PATH) | ADR-006 fallback advice |
| backends detected (per adapter `detect()`) | install hints |
| active-session lock consistent (no stale lock/orphan) | offer recovery |
| disk usage of store vs limits | prune hint |

---

## 14. Security model

### 14.1 Assets & trust boundaries

Assets: recorded session data (highest sensitivity), consent integrity, generated
skills (consumed by other agents), the user's shell environment, the local filesystem.

| # | Boundary | Expectations (normative) |
|---|---|---|
| B1 | zsh hook → ingest files | hook is fail-silent, append-only, writes only under the active session's `ingest/`; never `eval`s data; disabled instantly by pointer-file absence or `SKILLSMITH_DISABLE=1` |
| B2 | OS tool subprocesses (`screencapture`, `sips`, `osascript`, `git`, `tesseract`) | absolute/PATH-resolved binaries, argv arrays (no shell), timeouts, output caps; failures degrade recording, never crash the daemon |
| B3 | watched project files → store | file *content* is untrusted data: size caps before read, binary sniffing, no code execution based on content, symlinks not followed (`lstat` before read, skip symlinks) |
| B4 | session store at rest | `0700`/`0600`, single-user assumption documented; store path never derived from untrusted input; session ids validated by regex before path join |
| B5 | store → payload (redaction) | mandatory, fail-closed, covers every payload field (§9); payload hash recorded |
| B6 | payload → backend (consent) | exact-payload preview + explicit consent; only `redacted/` derivatives cross; consent persisted (§10) |
| B7 | backend subprocess | empty-cwd, minimal env, no shell, timeout, output caps; skillsmith holds no credentials (ADR-003) |
| B8 | backend output → filesystem | sentinel parser, path allowlist, size/count caps, no exec bits, no overwrite (§12.2, ADR-007) |
| B9 | config file & CLI args | strict schema, unknown keys rejected; `custom.argv` documented as "runs a command as you" with a doctor warning when set; config file `0600` |
| B10 | generated skill → downstream agents | dangerous-pattern lint warnings + `reviewed:false` provenance + documented human-review requirement (ADR-007) |

### 14.2 Abuse cases (must stay covered by issues 19, 20, 28, 29, 34)

1. Token visible in terminal screenshot → OCR → payload: redaction rules on OCR text;
   consent preview as human backstop.
2. Prompt injection on a visible web page steering skill content: ADR-007 output
   controls + dangerous-pattern lint + human review.
3. Malicious repo file names / gigantic files under a watched dir: ignore defaults,
   caps, symlink skip, path-length guard (B3).
4. Crafted backend output tries `../../.zshrc` write or exec-bit script: parser
   allowlist + no-exec (B8).
5. Another local user reads the store: 0700/0600 + documented single-user assumption (B4).
6. Screen share / meeting content of *other people* recorded: denylist + pause command +
  `PRIVACY.md` guidance; residual risk documented.
7. Malicious `customPatterns` regex (ReDoS) hanging redaction: regex compile timeout +
   per-rule execution time budget; on exceed → fail-closed with rule id named.

### 14.3 Secure defaults summary

Recording is opt-in per session; screenshots downscaled; denylist ships non-empty;
redaction cannot be disabled; consent interactive by default; scripts generation off;
no telemetry; no network I/O except the user's chosen backend CLI; store 0700.

---

## 15. Error handling, logging, testing, release

### 15.1 Errors & logging

Typed error hierarchy mapping to exit codes (§3.1). Logs to stderr, levels via
`--verbose/--quiet`; daemon logs to `daemon.log` with size cap (5 MiB, single rotation).
**Log redaction**: log formatter passes free-text fields through the redaction rules'
high-severity subset — logs must never contain raw captured secrets.

### 15.2 Testing strategy (full matrix in ISSUE_PLAN §Validation)

- Unit (vitest) per module; redaction rules require full vector coverage (every rule:
  ≥ 3 match, ≥ 3 no-match vectors).
- Integration: store lifecycle on tmpdir; ingest → normalize; digest goldens;
  parser/writer property-ish tests (path fuzz list); validator fixture corpus
  (valid/invalid skills).
- E2E (macOS runner): fixture session (canned events + tiny PNGs + canned OCR) →
  `redact` → `generate` with the `custom` fake backend (deterministic script) →
  `validate` → golden compare. Recorder E2E (real screencapture/OCR/zsh hook) runs
  locally via `npm run test:local` (documented; CI runners lack TCC).
- CI (GitHub Actions): lint + typecheck + unit/integration on `macos-14` and
  `ubuntu-latest` (pure-logic subset), E2E on macOS, `npm audit` gate (fail on high),
  build artifact.

### 15.3 Packaging & release

- `tsup` build → `dist/`, `bin.skillsmith`, `files` whitelist (dist, assets, LICENSE,
  README). `engines.node >= 20`. Zero postinstall scripts (supply-chain posture).
- Release: tag `v*` → GitHub Actions publish with **npm provenance** (OIDC trusted
  publishing; the owner configures the npm side manually — no tokens in repo secrets
  where avoidable). Changelog via Keep-a-Changelog `CHANGELOG.md`, versioning SemVer,
  `0.x` until capture formats stabilize.
- Install smoke test in CI: `npm pack` → global install into a temp prefix →
  `skillsmith --version`, `skillsmith validate` on a fixture.

---

## 16. Known unknowns (tracked in ISSUE_PLAN)

1. TCC attribution for the detached daemon's `screencapture` (issue 10 validation
   step on real hardware; fallback documented in issue 10).
2. vision-jxa OCR latency/accuracy (spike gate in issue 12; fallback flip per ADR-006).
3. Gemini CLI stdin piping behavior (issue 25 verification step).
4. `claude --bare` minimum version (issue 24 version-detect fallback).
5. chokidar performance on very large watched trees (issue 14 caps; possible v2
   watchman adapter).
6. Whether generated-skill quality needs terminal *output* capture (v2 PTY decision
   depends on v1 dogfooding).
7. FileVault-only at-rest posture acceptability for enterprise users (v2 encrypted
   store candidate).

## 17. Glossary

| Term | Meaning |
|---|---|
| session | one recorded work period, stored under `sessions/<id>/` |
| ingest file | raw append-only drop file written by the shell hook, normalized by the daemon |
| digest | redactable Markdown distillation of a session timeline |
| payload | the exact redacted bytes sent to a backend (after consent) |
| backend | an agent CLI adapter used for generation |
| run (`g-NNN`) | one generation attempt with its payload/consent/output/provenance |
| skill | output directory with `SKILL.md` per Agent Skills spec |
