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
| `/setup-rust-ci` | Scaffolds justfile + GHA workflows so local and CI run the same checks. |
| `/setup-crates-release` | Scaffolds a GHA release workflow that publishes to crates.io on tag. |
| `/audit-rust-plan <path>` | Scans an area, writes a ranked file-level audit checklist to `tmp/`. Cheap. |
| `/audit-rust-file [path]` | Deep file-level audit + refactor of one `.rs` file against the checklist. Expensive — run one file at a time. |

## File-level audit workflow

`/audit-rust-plan` and `/audit-rust-file` are a pair. The plan command is cheap
(structural scanning only) and produces a durable, resumable checklist under
`tmp/rust-audit-<slug>.md`. The file command is expensive (full read + refactor)
and is meant to be run once per file, ticking items off the checklist as you go.

Criteria it cares about: file header doc (`//!`), function length, nesting
depth, redundant comments, dead code, `unwrap` hygiene, visibility, layout,
and terse-but-meaningful docs. It stays inside one file — cross-file refactors
are flagged and deferred, not silently done.

## The rules

**The justfile is the source of truth for build operations.** GHA workflows
call `just` recipes. Local dev uses `just` recipes. The model uses `just`
recipes. Clippy flags, test scopes, and build targets are defined once, in
the justfile.

**The Rust toolchain is pinned in `rust-toolchain.toml` AND in every CI
workflow's `dtolnay/rust-toolchain@master` step.** Two files, one invariant:
they must agree. `/setup-rust-ci` enforces this on every run. Drift between
local and CI rustfmt becomes a loud failure instead of a silent `fmt-check`
failure on push.
