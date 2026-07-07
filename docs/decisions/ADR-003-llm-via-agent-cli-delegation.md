# ADR-003: LLM access via agent-CLI delegation; skillsmith holds no API keys

- Status: accepted
- Date: 2026-07-07
- Decision maker: repository owner (explicit confirmation), recorded by design agent

## Context

Synthesizing a SKILL.md from a session digest requires an LLM. Direct API integration
means skillsmith must manage credentials, retries, model differences, and would own a
credential-security surface in an OSS tool that also handles screen recordings.

## Decision

- skillsmith delegates generation to a **locally installed agent CLI** chosen by the
  user: `codex` (default), `claude`, `gemini`, or a user-configured custom command.
- Adapters spawn the CLI as a subprocess (argv array, no shell), in an empty temporary
  working directory, payload on stdin where supported
  (see `docs/research/agent-cli-backends.md`).
- **skillsmith never stores, reads, or forwards API keys.** Authentication is entirely
  the agent CLI's concern.
- A `custom` backend (explicit argv array in config) provides an escape hatch for other
  CLIs (e.g. local models via `ollama run`) and powers deterministic test fixtures.

## Consequences

- Zero credential-handling code → an entire vulnerability class is out of scope.
- Generation quality and cost tracking depend on the user's CLI setup; skillsmith only
  reports which backend/version produced an artifact (provenance).
- CLI flag drift is a maintenance risk → every adapter records the verified CLI version
  and re-verifies flags at implementation time (research doc requirement).
- The backend is a subprocess that may reach the network; the consent gate (ADR-004)
  is therefore positioned *before* the backend call, and the backend's output is treated
  as untrusted (ADR-007).

## Alternatives rejected

- Direct provider APIs: most control, but adds key management, provider matrices, and
  ongoing SDK churn to v1.
- Both in v1: doubles the surface before product validation; direct API adapters remain
  a v2 candidate behind the same `GenerationBackend` interface.
- No LLM (templates only): output quality too poor to be the product's core value.
