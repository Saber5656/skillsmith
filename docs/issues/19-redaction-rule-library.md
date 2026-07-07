# Title

Redaction rule library and entropy detector

## Summary

Implement `src/redact/patterns.ts` + `entropy.ts`: the built-in secret/PII detection
rules with test vectors, and the high-entropy token detector.

## Context

DESIGN §9.1 fixes the rule id list (stable public API); ADR-004 makes redaction
mandatory. This issue is detection only; application/masking is issue 20. Security
boundary B5 depends on the quality of this library.

## Scope

- Rule definitions + entropy detector + the full test-vector corpus. No engine, no CLI.

## Detailed Requirements

1. Rule shape:
   `{id, description, severity: 'high'|'medium'|'low', regex, postFilter?: (m) => boolean,
   testVectors: {match: string[], noMatch: string[]}}` — ids exactly:
   `aws-access-key-id, aws-secret-key, github-token, gitlab-pat, slack-token,
   stripe-key, openai-key, anthropic-key, google-api-key, jwt, pem-block,
   ssh-private-key, authorization-header, url-credentials, env-secret-assignment,
   generic-high-entropy, email` (17 rules; `ipv4-private` explicitly not shipped in v1).
2. Pattern requirements (representative anchors; implementer refines against vectors):
   - `aws-access-key-id`: `\b(AKIA|ASIA)[0-9A-Z]{16}\b` (high)
   - `github-token`: `\b(gh[pousr]_[A-Za-z0-9]{36,255}|github_pat_[A-Za-z0-9_]{22,255})\b` (high)
   - `gitlab-pat`: `\bglpat-[0-9A-Za-z_-]{20,}\b` (high)
   - `slack-token`: `\bxox[abposr]-[0-9A-Za-z-]{10,}\b` (high)
   - `stripe-key`: `\b[rs]k_live_[0-9a-zA-Z]{16,}\b` (high)
   - `openai-key`: `\bsk-(proj-)?[A-Za-z0-9_-]{20,}\b` with postFilter excluding
     matches also matching `anthropic-key` (medium — prefix collision risk)
   - `anthropic-key`: `\bsk-ant-[A-Za-z0-9-]{20,}\b` (high; must win over openai-key —
     rule order: anthropic before openai)
   - `google-api-key`: `\bAIza[0-9A-Za-z_-]{35}\b` (high)
   - `jwt`: `\beyJ[A-Za-z0-9_-]{8,}\.[A-Za-z0-9_-]{8,}\.[A-Za-z0-9_-]{8,}\b` (medium)
   - `pem-block`: `-----BEGIN [A-Z ]{0,32}PRIVATE KEY-----[\s\S]*?-----END [A-Z ]{0,32}PRIVATE KEY-----`
     multiline (high); `ssh-private-key`: OpenSSH header variant (high)
   - `authorization-header`: `(?i)\bauthorization\s*:\s*(bearer|basic|token)\s+\S{8,}` (high)
   - `url-credentials`: `\b[a-z][a-z0-9+.-]*://[^/\s:@]{1,64}:([^@/\s]{1,256})@` — capture
     group = password only (high; masking granularity defined in 20)
   - `env-secret-assignment`: `(?im)^\s*(?:export\s+)?([A-Z0-9_]*(SECRET|TOKEN|PASSWORD|PASSWD|API_KEY|APIKEY|PRIVATE_KEY|CREDENTIALS)[A-Z0-9_]*)\s*=\s*(?!["']?(\$\{|\$[A-Z_]))\S+`
     (high) — postFilter: value length ≥ 6 and not a placeholder
     (`xxx`, `changeme`, `<...>`, `${...}`)
   - `email`: RFC-lite `\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}\b` (low, PII
     category)
3. `generic-high-entropy` (medium): tokenizer splits on whitespace and common
   delimiters keeping `[A-Za-z0-9+/=_-]{20,}` candidates; Shannon entropy over the
   candidate > 4.0 bits/char AND ≥ 3 character classes AND not matching any allowlist
   or other rule → hit. Exclusions baked in: git SHAs (`^[0-9a-f]{7,40}$`),
   UUIDs, pure hex ≥ 32 (likely hashes — medium risk accepted, documented), file paths.
4. Vectors: every rule ≥ 3 `match` and ≥ 3 `noMatch` vectors, including the documented
   false-positive traps (git SHA vs entropy; `sk-ant` vs `sk-`; `Bearer xyz` short
   value). All secret-looking vectors must be **synthetic** (never real-looking leaked
   keys from corpora) and marked with a comment.
5. Rules compile once at module load; each regex must pass the 100 ms compile guard and
   a 10 ms/10 KiB-input execution budget on the vector corpus (ReDoS discipline, B5 +
   DESIGN §14.2 case 7).
6. Export `RULES: RedactionRule[]` (ordered: specific → generic; anthropic before
   openai; entropy last) and `HIGH_SEVERITY_RULES` subset (log sanitizer, issue 06/20).

## Acceptance Criteria

- Table-driven test executes every vector of every rule (match + noMatch) — 100 % pass.
- Rule-order test: `sk-ant-…` string hits `anthropic-key` only.
- Entropy detector: catches a synthetic 32-char base64 secret in prose; passes a git
  SHA, a UUID, an English sentence, a long file path.
- Performance test: full rule set over a 1 MiB synthetic digest < 2 s on CI.
- Public rule-id list snapshot test (API stability).

## Validation

`npm test`; reviewer manually spot-checks vectors for realism and synthetic-ness.

## Dependencies

01 (03/06 not required — pure module).

## Non-goals

Masking/application (20), NER-based PII detection (v2), image redaction (v2).

## Design References

- DESIGN §9.1, §14.2; ADR-004
