# Title

Project scaffolding and TypeScript toolchain

## Summary

Create the npm package skeleton for the `skillsmith` CLI: strict TypeScript, build,
lint/format, test runner, license, and repository hygiene files. After this issue the
repo builds a runnable (stub) binary.

## Context

The repository currently contains only `README.md`. Every other issue assumes the
toolchain defined here. Platform/language decisions are fixed by ADR-002
(macOS-first, TypeScript, Node >= 20, npm distribution) and ADR-008 (MIT).

## Scope

- `package.json`, `tsconfig.json`, ESLint + Prettier, Vitest, tsup build, `LICENSE`,
  `.gitignore`, `.editorconfig`, npm scripts, stub CLI entry.
- No product features; no CI (issue 02).

## Detailed Requirements

1. `package.json`:
   - `name: "skillsmith"`, `version: "0.0.0"`, `license: "MIT"`, `type: "module"`,
     `engines: {"node": ">=20"}`, `bin: {"skillsmith": "dist/cli/index.js"}`,
     `files: ["dist", "assets", "LICENSE", "README.md"]`.
   - Scripts: `build` (tsup), `dev` (tsup --watch), `lint` (eslint .),
     `format` / `format:check` (prettier), `typecheck` (tsc --noEmit),
     `test` (vitest run), `test:watch`, `test:local` (placeholder that echoes
     "local-only suite; see issue 35" and exits 0).
   - **No `postinstall` or other lifecycle scripts** (DESIGN §15.3).
2. `tsconfig.json`: `strict: true`, `noUncheckedIndexedAccess: true`,
   `module: "NodeNext"`, `moduleResolution: "NodeNext"`, `target: "ES2022"`,
   `outDir: "dist"`, `rootDir: "src"`.
3. tsup config: entry `src/cli/index.ts`, format `esm`, `banner` adding
   `#!/usr/bin/env node` to the CLI entry, sourcemaps on.
4. ESLint flat config with `@typescript-eslint` recommended-type-checked; Prettier
   defaults (100-col print width); both wired to the scripts above.
5. Vitest config: `test/**/*.test.ts` + `src/**/*.test.ts`, coverage provider v8
   (thresholds set later).
6. Stub entry `src/cli/index.ts`: prints `skillsmith <version> (design-phase stub)` and
   exits 0; reads version from `package.json` at build time (tsup define or import).
7. `LICENSE`: MIT, copyright `2026 Saber5656` (ADR-008).
8. `.gitignore`: `node_modules/`, `dist/`, `coverage/`, `*.tsbuildinfo`, `.DS_Store`.
9. Lockfile: commit `package-lock.json` (npm, not pnpm/yarn — single supported flow).

## Acceptance Criteria

- `npm ci && npm run build` succeeds on Node 20 with no warnings that fail the build.
- `node dist/cli/index.js` prints the stub line and exits 0.
- `npm run lint`, `npm run typecheck`, `npm run format:check`, `npm test` all pass
  (a trivial placeholder test exists so vitest exits 0).
- `npm pack --dry-run` lists only `dist/`, `assets/` (may be absent), `LICENSE`,
  `README.md`, `package.json`.
- No lifecycle install scripts in `package.json`.

## Validation

Run the five commands above locally and paste output into the PR description. Verify
the packed tarball file list.

## Dependencies

None (first issue).

## Non-goals

CI workflows (02), any CLI commands (06+), publishing (37).

## Design References

- DESIGN §2 (module map), §15.3 (packaging)
- ADR-002, ADR-008
