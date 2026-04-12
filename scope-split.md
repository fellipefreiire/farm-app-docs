# Scope Split — contextzip (TokenZip) vs context-mode

Both tools intercept tool output before it enters the context window. To prevent
hook collision and double-filtering, each command has exactly **one owner**.

## Ownership table

| Tool / operation | Owner | Reason |
|---|---|---|
| `pnpm *` (any subcommand) | **contextzip** | Specialized filter, strips boilerplate |
| `pnpm test` / `vitest` / `vitest run` | **contextzip** | `--only-failures`-style filter |
| `pnpm test:e2e` / `playwright test` | **contextzip** | `playwright` filter |
| `tsc` / `pnpm tsc` / `pnpm build` (typecheck) | **contextzip** | Grouped errors per file |
| `next build` / `next dev` | **contextzip** | Compact build output |
| `lint` / `eslint` / `pnpm lint` | **contextzip** | Grouped rule violations |
| `prettier` / `prettier --check` / `format` | **contextzip** | Compact format check |
| `prisma *` (`generate`, `migrate dev`, `studio`) | **contextzip** | Strip ASCII art |
| `git *` (`status`, `log`, `diff`, `show`) | **contextzip** | Compact by default |
| `gh *` (any `gh` CLI call) | **contextzip** | Token-optimized GH output |
| `grep` / `rg` (direct shell, not the Grep tool) | **contextzip** | Groups by file, strips whitespace |
| `find` / `ls` / `tree` / `wc` | **contextzip** | Token-optimized traversal |
| `curl` / `wget` / `web` extraction | **contextzip** | Auto-JSON schema / strip nav-ads |
| `docker *` / `kubectl *` | **contextzip** | Compact |
| `psql` | **contextzip** | Strip borders, compress tables |
| `Read` (Claude Code built-in tool) | **context-mode** | Generic sandbox on large file reads |
| `WebFetch` (Claude Code built-in tool) | **context-mode** | Generic sandbox on fetched HTML/text |
| `Grep` (Claude Code built-in tool) | **context-mode** | Generic sandbox |
| `Glob` (Claude Code built-in tool) | **context-mode** | Generic sandbox |
| `Bash` commands **not** listed above | **context-mode** | Fallback sandbox |
| Session state (restore after break) | **context-mode** | SQLite FTS5, no equivalent in contextzip |
| Per-tool token breakdown (`ctx-stats`) | **context-mode** | Tokens-centric measurement |
| Cumulative savings ledger (`gain`) | **contextzip** | Savings-centric measurement, complementary |
| $ cost tracking (`cc-economics`) | **contextzip** | Integrates with `ccusage`, complementary |
| Retroactive savings scan (`discover`) | **contextzip** | Unique capability |

## Measurement stack (non-conflicting)

| Question | Tool |
|---|---|
| How many tokens did this session cost, per tool? | `/context-mode:ctx-stats` |
| How much have I saved cumulatively? | `contextzip gain` |
| What did it cost in $ and what was the savings ratio? | `contextzip cc-economics` |
| Where am I leaving savings on the table? | `contextzip discover` |

All four are safe to run — they read logs/state, they don't mutate hooks.

## Verification after install

```
contextzip verify            # validates hook integrity + inline TOML tests
contextzip hook-audit        # rewrite audit (needs TOKENZIP_HOOK_AUDIT=1)
/context-mode:ctx-stats      # sanity check context-mode is active
```

Run these after Step 0 of `_BASELINE.md` to confirm both tools are wired and the
scope split holds.

## What to do if a collision shows up

Symptoms: output gets double-filtered (truncated twice), or one tool's filter
silently drops something the other expected to see.

1. Run `contextzip hook-audit` and read the rewrite log.
2. Identify which tool rewrote the command.
3. Check this table — is the owner correct?
4. If contextzip is intercepting a command that should fall through to
   context-mode, add an exception via project-local TOML filter
   (`contextzip trust` then edit).
5. If context-mode is intercepting a command that contextzip already filtered,
   the context-mode hook needs to defer — check the plugin config.

**Never fix a collision by disabling both hooks for a command.** If neither owns
it, raw output enters the context and the savings vanish.
