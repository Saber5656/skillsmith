# ADR-004: mandatory redaction pass and explicit consent gate before any external send

- Status: accepted
- Date: 2026-07-07
- Decision maker: repository owner (explicit confirmation), recorded by design agent

## Context

Recorded sessions inevitably contain secrets and PII: tokens echoed in terminals,
`.env` contents in diffs, passwords and emails visible in screenshots (OCR text),
credentials embedded in URLs. The generation step hands session-derived text to an agent
CLI that typically forwards it to a cloud LLM. This is the product's single most
dangerous data flow.

## Decision

1. **Redaction is mandatory and fail-closed.** `skillsmith generate` refuses to build a
   payload unless the redaction engine ran successfully over every payload field
   (shell commands, OCR text, diffs, window titles, paths, notes). There is no flag to
   skip redaction.
2. **Consent is explicit and payload-exact.** Before invoking the backend, skillsmith
   shows the exact redacted payload (pager) plus a redaction report summary, and
   requires an interactive confirmation. The consent record (payload SHA-256, backend
   id, model, timestamp, app version) is persisted with the generation run.
3. **Non-interactive consent is opt-in twice**: `--yes` is honored only when the config
   sets `consent.allowNonInteractive: true`. Default is `false`.
4. Redaction uses irreversible masking with stable per-session placeholders
   (`[REDACTED:<rule-id>:<n>]`) so structure remains learnable while values are gone.
5. Originals are never modified; redacted derivatives live in a separate directory.

## Consequences

- One extra UX step per generation — accepted cost for a safe OSS default.
- Redaction can never be perfect (OCR text especially); consent-with-preview keeps the
  human as the last check, and the frontmost-app denylist (ADR-001) reduces exposure at
  capture time.
- Every payload is reproducible and auditable via its hash + consent record.

## Alternatives rejected

- Automatic redaction without consent: silent leakage of missed secrets.
- Local-model-only default: strongest guarantee but couples default output quality to
  local model performance; remote backends are the product's realistic default.
