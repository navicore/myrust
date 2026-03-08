# myrust

Rust-specific conventions for AI coding tools. Companion to
[myspec](https://github.com/navicore/myspec) — install both.

## The problem

AI coding tools default to raw `cargo` commands. This bypasses your justfile,
misses clippy flags, skips CI-equivalent checks, and creates drift between local
dev and GHA workflows. Then CI fails on push and you waste a cycle.

## Install

```zsh
mkdir -p .claude/commands && cp -R ~/git/navicore/myrust/.claude/commands/* .claude/commands/
```

## Commands

| Command | What it does |
|---------|-------------|
| `/just-check` | Verifies justfile is source of truth — no raw cargo in GHA workflows. |
| `/just-run <recipe>` | Runs a just recipe. Reminds the model to never use raw cargo. |
| `/ci-check` | Runs `just ci` — the same checks GHA will run. |

## The rule

**The justfile is the source of truth.** GHA workflows call `just` recipes.
Local dev uses `just` recipes. The model uses `just` recipes. Clippy flags,
test scopes, and build targets are defined once, in the justfile.
