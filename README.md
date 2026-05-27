# bun-elision-repro

Minimal repro: `bun run --filter` ignores `[run] elide-lines` in `bunfig.toml`.
The value is not just clamped or defaulted — the entire `[run]` table is not
read under `--filter`.

## Setup

- Workspace root with `bunfig.toml` containing `[run] elide-lines = <N>`
- `packages/app/log-many.ts` prints 50000 lines + `---END---` sentinel
- Tested on `bun --version` = 1.3.14 (stable), 1.4.0 (canary), and earlier versions back to 1.1.17

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
| H' | `bun run --config ./bunfig.toml --filter app print-many` (space form) | `elide-lines = 0` | **silently produces no output (0 bytes)** |
| I | `BUN_CONFIG_PATH=$PWD/bunfig.toml bun run --filter ...` | env path | YES |
| J | bunfig top-level `elide-lines = 0` (no `[run]` table) | malformed | parse error |
| K | `bun run --filter app print-many` with `elide-lines = 5` | `elide-lines = 5` | YES, **10-line chunks (default)** |
| L | same with `elide-lines = 50` | `elide-lines = 50` | YES, **10-line chunks** |
| M | same with `elide-lines = 1000` | `elide-lines = 1000` | YES, **10-line chunks** |

## Conclusion

`bun run --filter` does **not** read `bunfig.toml` at all. Across `elide-lines`
values 0, 5, 10, 50, 1000 the behavior is identical — every run uses the
hardcoded default of 10 lines per chunk. So this is not a "value is clamped"
bug; the `[run]` table is simply never consulted under `--filter`.

The TOML parser is fine — `--config=./bunfig.toml` loads the same file
successfully (row H). The bug is specifically in `--filter`'s auto-discovery.

Persists in 1.1.17 → 1.4.0 canary (never worked). Independently verified
by Codex on the same machine.

**Side bug (row H'):** `--config <path>` with a space (instead of `=`) under
`--filter` silently produces zero bytes (exit 0, no output, no error). Only
`--config=<path>` actually loads the config.

## Upstream issues

- <https://github.com/oven-sh/bun/issues/17918> — OPEN (exact match)
- <https://github.com/oven-sh/bun/issues/12183> — OPEN (filter vs cd divergence)
- <https://github.com/oven-sh/bun/issues/21764> — CLOSED (older variant)

## Workarounds

1. **Per-script flag** — hardcode `--elide-lines=0` in `package.json` script (ugly, every script).
2. **Wrapper command** — shell script around `bun run` that injects `--elide-lines=0`.
3. **Shell alias** — `alias brun='bun run --elide-lines=0'`. Local only.
4. **Explicit `--config=`** — `bun run --config=./bunfig.toml --filter ...`. Bunfig honored.
5. **Direct cd** — `cd packages/app && bun run print-many`. Bunfig honored.

## Reproduce

```bash
git clone https://github.com/biteinc/bun-elision-repro.git
cd bun-elision-repro
bun install

# Elision messages only appear when stdout is a TTY.
# script(1) on macOS needs a real terminal, so capture via expect(1):
cat > /tmp/runbun.exp <<'EOF'
#!/usr/bin/expect -f
set timeout 60
log_file -noappend [lindex $argv 0]
spawn -noecho bun run --filter app print-many
expect eof
EOF
chmod +x /tmp/runbun.exp
expect /tmp/runbun.exp /tmp/out.log > /dev/null
grep -c 'elided' /tmp/out.log  # > 0 = bug present
```

To confirm the value is ignored, change `elide-lines` in `bunfig.toml` to
any value (5, 50, 1000) and rerun — chunks stay at 10 lines.
