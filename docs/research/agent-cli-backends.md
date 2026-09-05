# Research: agent CLI backends for non-interactive generation

- Status: verified (invocation surfaces); flags may drift — each adapter issue includes a re-verification step
- Verified on: 2026-07-07
- Sources:
  - Codex: <https://developers.openai.com/codex/noninteractive> (fetched)
  - Claude Code: <https://code.claude.com/docs/en/headless> (fetched)
  - Gemini CLI: <https://github.com/google-gemini/gemini-cli> README (fetched)
- Impact: defines adapter contracts for issues 22–26 and the security posture of ADR-003

## Requirement

skillsmith delegates SKILL.md synthesis to a locally installed agent CLI. The adapter
must run the CLI: (a) non-interactively, (b) with tools/side effects minimized,
(c) in an empty working directory, (d) with the payload passed via stdin where possible,
and (e) with machine-parsable output.

## Codex CLI (`codex`)

| Aspect | Verified surface |
|---|---|
| Non-interactive command | `codex exec "<prompt>"` |
| Prompt via stdin | `codex exec -` reads the full prompt from stdin |
| Final answer to file | `-o <path>` / `--output-last-message <path>` writes final message to a file |
| JSON stream | `--json` (JSON Lines on stdout) |
| Structured output | `--output-schema <path>` |
| Sandbox | Default is read-only for `exec`; `--sandbox workspace-write` / `--sandbox danger-full-access` opt in to more access |
| Session persistence | `--ephemeral` avoids persisting rollout files (privacy-positive for skillsmith) |
| Git check | `--skip-git-repo-check` needed when cwd is not a git repo (skillsmith uses an empty temp dir) |
| Config isolation | `--ignore-user-config` skips `$CODEX_HOME/config.toml` (not used by default: would drop the user's model/auth setup) |

Planned invocation (issue 23):

```
codex exec --ephemeral --skip-git-repo-check -o <tmpdir>/last-message.md -
# payload on stdin, cwd = empty 0700 temp dir, result read from last-message.md
```

## Claude Code CLI (`claude`)

| Aspect | Verified surface |
|---|---|
| Non-interactive | `claude -p "<prompt>"` (`--print`) |
| Deterministic scripted mode | `--bare` skips hooks, skills, plugins, MCP servers, CLAUDE.md discovery — recommended for scripts |
| Stdin | Piped stdin is read as input data; capped at 10 MB (v2.1.128+), exceeding it exits non-zero |
| Output | `--output-format json` → `.result` field holds text; also `stream-json` |
| Schema-constrained output | `--output-format json --json-schema '<schema>'` → `.structured_output` |
| Tool restriction | `--permission-mode dontAsk` denies anything not explicitly allowed; `--allowedTools` allowlists |
| System prompt | `--append-system-prompt` |

Planned invocation (issue 24):

```
claude --bare -p "<instructions>" --permission-mode dontAsk --output-format json
# payload piped on stdin (< 10 MB enforced by skillsmith's payload budget), parse .result
```

Note: `--bare` availability depends on Claude Code version; adapter must version-detect
and fall back to plain `-p` with the same permission mode.

## Gemini CLI (`gemini`)

| Aspect | Verified surface |
|---|---|
| Non-interactive | `gemini -p "<prompt>"` |
| Output | `--output-format json` (structured), `stream-json` (NDJSON events) |
| Model | `-m <model>` |
| Stdin piping | Not confirmed in fetched README — **must be verified during implementation** (issue 25); fallback is embedding the payload in the `-p` argument with an OS `ARG_MAX` guard |
| Sandbox/approval flags | Not confirmed in fetched README — verify during implementation |

## Cross-cutting adapter rules (issue 22)

- Spawn with `execFile` semantics (argv array, **never** a shell string).
- `cwd` = freshly created empty temp directory, mode `0700`, removed afterwards.
- Environment: pass-through minimal set (`PATH`, `HOME`, locale); strip `SKILLSMITH_*`.
  `HOME` is required because all three CLIs read their auth/config from the home directory.
- Timeout (default 300 s) with process-group kill; stdout/stderr capture capped (10 MB).
- skillsmith stores **no API keys**; authentication is wholly owned by the agent CLI.
- Treat all CLI output as untrusted input (see ADR-007).

## Known risk

All three CLIs evolve quickly. Each adapter issue carries an acceptance criterion:
"re-verify the documented flags against the installed CLI version and record the
verified version in the adapter's source comment block".
