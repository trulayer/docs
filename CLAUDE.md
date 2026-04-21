# CLAUDE.md — Developer Documentation

Public TruLayer developer docs, hosted on Mintlify at docs.trulayer.ai. See `AGENTS.md` for writing conventions and `TASKS.md` for work in flight.

## Key Commands

```bash
pnpm dev            # mint dev — local preview on localhost:3000
pnpm broken-links   # mint broken-links — scan for dead internal links
pnpm sync-openapi   # pull backend OpenAPI spec into api-reference/
```

`mint` is available globally (installed via Homebrew) and also declared as a devDependency for CI.

## Claude Preview

`.claude/launch.json` wires the dev server so Claude Preview can boot it for visual verification. Runs `pnpm dev` on port 3000.

## Node 25.9 localStorage shim

If running on Node 25.9 (Homebrew default as of April 2026), any Node-backed dev server (including downstream sibling repos) crashes with `TypeError: localStorage.getItem is not a function`. Node 25.9 ships an experimental `globalThis.localStorage` that only initializes with `--localstorage-file=<path>`.

A process-wide shim at `~/.claude/shims/no-localstorage.cjs` is auto-loaded via `NODE_OPTIONS` in `~/.zshrc`. Verify with:

```sh
echo "$NODE_OPTIONS"
# expect: --require /Users/<you>/.claude/shims/no-localstorage.cjs
```

Mintlify itself isn't affected, but keeping the note here so the shim is discoverable from any sibling repo.
