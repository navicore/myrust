Set up GitHub Actions CI for a Rust project using `just` as the single source of truth for build/test/lint. Both local dev and CI call the same recipes — no drift. $ARGUMENTS

## Inputs

1. Read `Cargo.toml` to find: workspace members (if any), edition, toolchain requirements
2. Read `justfile` if it exists — adapt existing recipes rather than replacing them
3. Read `.github/workflows/` if it exists — understand what's already configured
4. Check `rust-toolchain.toml` or existing CI for the pinned Rust version, or default to stable

## Steps

### 1. Create or update the justfile

The justfile is the **source of truth** for all build operations. CI workflows call `just ci` — nothing else.

Required recipes:
- `fmt`: `cargo fmt --all`
- `fmt-check`: `cargo fmt --all -- --check`
- `lint`: `cargo clippy --workspace --all-targets -- -D warnings` (warnings are errors)
- `test`: `cargo test --workspace --all-targets`
- `build`: `cargo build --release` (adapt for workspace members that need special handling)
- `ci`: chains `fmt-check lint test build` — the single command developers run before pushing

The `ci` recipe should print a summary on success:
```
@echo "Safe to push to GitHub - CI will pass."
```

### 2. Create CI workflows

Strategy: Linux on PRs (fast feedback), macOS on merge to main (saves cost).

Create `.github/workflows/ci-linux.yml` (triggers on `pull_request` to main):
- Use `extractions/setup-just@v2` to install just
- Use `dtolnay/rust-toolchain@master` with pinned toolchain version + `rustfmt, clippy` components
- Cache `~/.cargo/registry` and `~/.cargo/git` keyed by `Cargo.lock` hash
- Do NOT cache `target/` for test runs (stale test binaries cause false failures)
- Run `just ci` as the single CI step
- Add comment: "This workflow ONLY calls `just` commands. The justfile is the source of truth."

Create `.github/workflows/ci-macos.yml` (triggers on `push` to main):
- Same structure as Linux but `runs-on: macos-latest`
- Skip system dependency installation that's macOS-preinstalled

### 3. Add CI section to justfile header

```just
# Run all CI checks (same as GitHub Actions!)
# This is what developers should run before pushing
ci: fmt-check lint test build
```

## Constraints

- Do NOT modify Rust source code, Cargo.toml dependencies, or library code
- Do NOT change the project's Rust edition or toolchain version — read what's configured
- Do NOT add release, publishing, benchmarking, or security scanning — those are separate concerns
- If a justfile already exists, preserve existing recipes and add missing ones
- If CI workflows already exist, update them to use `just ci` rather than duplicating build logic

## Final step

Update relevant docs (README.md or ARCHITECTURE.md) to document:
- How to run CI locally: `just ci`
- What CI checks: formatting, clippy (warnings=errors), tests, build

<!-- Reference: pattern extracted from navicore/patch-seq -->
