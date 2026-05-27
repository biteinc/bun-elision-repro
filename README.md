# bun-elision-repro

Minimal repro: `bun run --filter` ignores `[run] elide-lines` in `bunfig.toml`.

## Reproduce

```bash
git clone https://github.com/biteinc/bun-elision-repro.git
cd bun-elision-repro
bun install
```

Bug — elides despite `bunfig.toml` saying `[run] elide-lines = 0`:

```bash
bun run --filter app print-many
```

Expected: 50000 lines + `---END---`.
Actual: chunks of 10 lines with `[N lines elided]` between them.

Workaround — passing the flag explicitly works:

```bash
bun run --elide-lines=0 --filter app print-many
```

Now you see all 50000 lines.

## What's broken

`bun run --filter` does **not** read `bunfig.toml`. The value of
`elide-lines` doesn't matter — set it to 0, 5, 50, 1000 and the output
is identical (10-line chunks, the hardcoded default). The whole `[run]`
table is never consulted under `--filter`.

The TOML parser is fine: `--config=./bunfig.toml` loads the same file
and works (see matrix row H below). The bug is specifically in
`--filter`'s auto-discovery.

Persists in bun 1.1.17 → 1.4.0 canary (never worked). Independently
verified by Codex on the same machine.

## Side bug

`bun run --config <path> --filter ...` (space between `--config` and
the path) silently produces 0 bytes — exit 0, no output, no error.
Only `--config=<path>` (with `=`) actually loads the config.

## Test matrix

| # | Command | Bunfig | Elides? |
|---|---------|--------|---------|
| A | `bun run --filter app print-many` | `elide-lines = 0` | **YES (bug)** |
| B | `bun run --elide-lines=0 --filter app print-many` | `elide-lines = 0` | NO |
| C | `cd packages/app && bun run print-many` | `elide-lines = 0` | NO |
| D | A but with different script name | `elide-lines = 0` | YES |
| E | A but with bunfig also in package dir | root + pkg | YES |
| F | `BUN_RUN_ELIDE_LINES=0 bun run --filter ...` (env var, not documented) | `elide-lines = 0` | YES |
| G | A on bun 1.4.0 canary | `elide-lines = 0` | YES |
| H | `bun run --config=./bunfig.toml --filter app print-many` | `elide-lines = 0` | NO |
| H' | `bun run --config ./bunfig.toml --filter app print-many` (space form) | `elide-lines = 0` | **silently 0 bytes** |
| I | `BUN_CONFIG_PATH=$PWD/bunfig.toml bun run --filter ...` | env path | YES |
| J | bunfig top-level `elide-lines = 0` (no `[run]` table) | malformed | parse error |
| K | A with `elide-lines = 5` | `elide-lines = 5` | YES, **10-line chunks (default)** |
| L | A with `elide-lines = 50` | `elide-lines = 50` | YES, **10-line chunks** |
| M | A with `elide-lines = 1000` | `elide-lines = 1000` | YES, **10-line chunks** |

## Workarounds

1. **Per-script flag** — hardcode `--elide-lines=0` in `package.json` script (ugly, every script).
2. **Wrapper command** — shell script around `bun run` that injects `--elide-lines=0`.
3. **Shell alias** — `alias brun='bun run --elide-lines=0'`. Local only.
4. **Explicit `--config=`** — `bun run --config=./bunfig.toml --filter ...`. Bunfig honored.
5. **Direct cd** — `cd packages/app && bun run print-many`. Bunfig honored.

## Upstream issues

- <https://github.com/oven-sh/bun/issues/17918> — OPEN (exact match)
- <https://github.com/oven-sh/bun/issues/12183> — OPEN (filter vs cd divergence)
- <https://github.com/oven-sh/bun/issues/21764> — CLOSED (older variant)
