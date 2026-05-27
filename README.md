# bun-elision-repro

Minimal repro: `bun run --filter` ignores `[run] elide-lines = 0` in `bunfig.toml`.

## Setup

- Workspace root with `bunfig.toml` containing `[run] elide-lines = 0`
- `packages/app/log-many.ts` prints 50000 lines + `---END---` sentinel
- `bun --version` = 1.3.14 (stable), 1.4.0 (canary)

## Test matrix

| # | Command | Bunfig | Elides? |
|---|---------|--------|---------|
| A | `bun run --filter app print-many` | root | **YES (bug)** |
| B | `bun run --elide-lines=0 --filter app print-many` | root | NO |
| C | `cd packages/app && bun run print-many` | root | NO |
| D | A but with different script name | root | YES |
| E | A but with bunfig also in package dir | root + pkg | YES |
| F | `BUN_RUN_ELIDE_LINES=0 ...` (env var, not documented) | root | YES |
| G | A on bun 1.4.0 canary | root | YES |
| H | `bun run --config=./bunfig.toml --filter app print-many` | root via `--config=` | NO |
| H' | `bun run --config ./bunfig.toml --filter app print-many` (space form) | root via `--config` | **silently produces no output** |
| I | `BUN_CONFIG_PATH=$PWD/bunfig.toml bun run --filter ...` | env path | YES |
| J | bunfig top-level `elide-lines = 0` (no `[run]` table) | malformed | parse error |

## Conclusion

`--filter` execution path does **not** auto-load `bunfig.toml`.
Workarounds: `--elide-lines=N` CLI flag, OR explicit `--config=./bunfig.toml` (use `=` form).
TOML parser itself works fine — only auto-discovery under `--filter` is broken.
Bug persists in bun 1.4.0 canary. Independently verified by Codex on the same machine.

**Side bug:** `--config <path>` (space form) under `--filter` silently swallows all output (exit 0, zero bytes). Only `--config=<path>` form actually loads the config.

## Upstream issues

- <https://github.com/oven-sh/bun/issues/17918> — OPEN (exact match)
- <https://github.com/oven-sh/bun/issues/12183> — OPEN (filter vs cd divergence)
- <https://github.com/oven-sh/bun/issues/21764> — CLOSED (older variant)

## Workarounds

1. **Per-script flag** — hardcode `--elide-lines=0` in package.json script (ugly, every script).
2. **Wrapper command** — shell script around `bun run` that injects `--elide-lines=0`.
3. **Shell alias** — `alias brun='bun run --elide-lines=0'`. Local only.
4. **Direct cd** — `cd packages/app && bun run print-many`. Bunfig honored.

## Reproduce

```bash
git clone <this>
cd bun-elision-repro
bun install

# Elision messages only appear when stdout is a TTY, so capture via script(1):
script -q /tmp/bun-A.log bash -c 'bun run --filter app print-many' > /dev/null 2>&1
grep -c 'elided' /tmp/bun-A.log  # > 0 = bug present
```
