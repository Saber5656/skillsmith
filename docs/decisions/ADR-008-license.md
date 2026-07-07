# ADR-008: MIT license

- Status: accepted (trivially reversible before first release)
- Date: 2026-07-07
- Decision maker: design agent (conservative default; flag to owner before `v0.1.0` tag)

## Context

The repository is intended to become a public OSS project (release posture set by the
task owner). A license must exist before the first public release and before accepting
external contributions.

## Decision

MIT license, copyright holder = repository owner (`Saber5656`).

## Consequences

- Maximum adoption simplicity; matches the dominant convention for developer CLI tools
  in the npm ecosystem.
- No patent grant (unlike Apache-2.0). skillsmith contains no novel patented mechanisms;
  risk accepted for v1.
- `LICENSE` file is created in issue 01; `package.json` sets `"license": "MIT"`.

## Alternatives rejected

- Apache-2.0: explicit patent grant and NOTICE mechanics; heavier than needed here.
  Revisit only if the owner requests it before the first tagged release.
