# Title

Path resolution and YAML config loader with strict schema

## Summary

Implement `src/config/`: XDG-aware path resolution, the full v1 config schema (zod),
YAML loading with defaults and strict unknown-key rejection, and the
`skillsmith config show|path` commands.

## Context

Every subsystem reads configuration through this module. The complete schema, defaults,
and precedence are normative in DESIGN §4; storage paths in DESIGN §5.1 and ADR-005.

## Scope

- `src/config/paths.ts`, `src/config/schema.ts`, `src/config/loader.ts`,
  `src/cli/commands/config.ts`. Config **writing** is out of scope except `init`'s
  scaffold (issue 32).

## Detailed Requirements

1. `paths.ts` exports:
   - `configDir()` → `$XDG_CONFIG_HOME/skillsmith` else `~/.config/skillsmith`
   - `dataDir()` → `$XDG_DATA_HOME/skillsmith` else `~/.local/share/skillsmith`
   - `configFile()`, `sessionsDir()`, `activeSessionFile()`, `activeSessionLock()`
   - `SKILLSMITH_CONFIG` env var overrides `configFile()` (absolute path required).
2. `schema.ts`: zod schema implementing **exactly** the YAML structure, field names,
   defaults, and ranges from DESIGN §4 (copy the table; ranges like
   `intervalSeconds: 5..300`, `maxWidthPx: 800..4000` are `.min()/.max()`).
   `version` must equal 1. Unknown keys anywhere → error (`.strict()` on every object)
   including a did-you-mean hint using Levenshtein distance ≤ 2 against sibling keys.
3. `loader.ts`:
   - `loadConfig(): {config, sourcePath | null, warnings[]}`; absent file → all defaults.
   - Parse with `yaml` package; YAML errors reported with line/column.
   - `redaction.customPatterns[].regex` compiled at load; invalid regex → load error
     naming the pattern id. Compilation must apply a 100 ms compile budget guard
     (catastrophic patterns rejected; DESIGN §14.2 case 7).
   - If the config file mode is group/world-readable, emit a warning (fixed by doctor).
4. CLI:
   - `skillsmith config path` prints the effective path (even if absent, with
     `(not created)` suffix on stderr in human mode).
   - `skillsmith config show [--json]` prints effective config (defaults merged);
     `--json` prints canonical JSON. Never prints `customPatterns` regexes flagged
     invalid — load fails before display.
5. Errors map to exit 1 with a `Config error:` prefix (exit-code framework lands in 06;
   until then, throw typed `ConfigError`).

## Acceptance Criteria

- Unit tests cover: default-only load; full-file load; unknown key rejection with hint;
  range violations; invalid custom regex; XDG override; `SKILLSMITH_CONFIG` override;
  version mismatch.
- `skillsmith config show` output for an empty system matches the DESIGN §4 defaults
  exactly (golden test).
- Loading a config with `consent.allowNonInteractive: true` round-trips (needed by 21).

## Validation

`npm test` green; run `config show`/`config path` manually with and without
`XDG_CONFIG_HOME` set and paste outputs.

## Dependencies

01, 06 (CLI wiring can land as a follow-up commit within this issue if 06 is not merged;
module API must not depend on CLI).

## Non-goals

Writing/scaffolding config files (32), config migration (v2).

## Design References

- DESIGN §4, §5.1, §14.1 B9; ADR-005
