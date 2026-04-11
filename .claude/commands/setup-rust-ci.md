Set up GitHub Actions CI for a Rust project using `just` as the single source of truth for build/test/lint. Both local dev and CI call the same recipes — no drift. $ARGUMENTS

## Inputs

1. Read `Cargo.toml` to find: workspace members (if any), edition, toolchain requirements
2. Read `justfile` if it exists — adapt existing recipes rather than replacing them
3. Read `.github/workflows/` if it exists — understand what's already configured
4. Check the local Rust version (`rustc --version`) — this version is written to `rust-toolchain.toml` AND to the `toolchain:` input of every CI workflow. Both sources must agree.
5. Verify `Cargo.lock` exists and is tracked by git — if not, error out and tell the user to commit it

## Steps

### 1. Create or update the justfile

The justfile is the **source of truth** for all build operations. CI workflows call `just ci` — nothing else.

Required recipes (all cargo commands except `fmt` must use `--locked` to enforce Cargo.lock):
- `fmt`: `cargo fmt --all`
- `fmt-check`: `cargo fmt --all -- --check`
- `lint`: `cargo clippy --locked --workspace --all-targets -- -D warnings` (warnings are errors)
- `test`: `cargo test --locked --workspace --all-targets`
- `build`: `cargo build --locked --release` (adapt for workspace members that need special handling)
- `ci`: chains `fmt-check lint test build` — the single command developers run before pushing

The `ci` recipe should print a summary on success:
```
@echo "Safe to push to GitHub - CI will pass."
```

### 2. Create CI workflows

Strategy: Linux on PRs (fast feedback), macOS on merge to main (saves cost).

Create `.github/workflows/ci-linux.yml` (triggers on `pull_request` to main):
- Use `extractions/setup-just@v2` to install just
- Use `dtolnay/rust-toolchain@master` with `toolchain: <version from step 4>` + `components: rustfmt, clippy`. Note: `dtolnay/rust-toolchain` does NOT auto-read `rust-toolchain.toml` — the `toolchain:` input is required. The same version also lives in `rust-toolchain.toml` (see 2a) and the two MUST agree; 2b enforces this.
- Do NOT use `@stable` — it drifts between runs and causes CI failures that can't be reproduced locally
- Cache `~/.cargo/registry` and `~/.cargo/git` keyed by `Cargo.lock` hash
- Do NOT cache `target/` for test runs (stale test binaries cause false failures)
- Run `just ci` as the single CI step
- Add comment: "This workflow ONLY calls `just` commands. The justfile is the source of truth."

Create `.github/workflows/ci-macos.yml` (triggers on `push` to main):
- Same structure as Linux but `runs-on: macos-latest`
- Same pinned toolchain version — must match `rust-toolchain.toml` and the Linux workflow exactly
- Skip system dependency installation that's macOS-preinstalled

### 2a. Create `rust-toolchain.toml`

Write at the repo root:

```toml
[toolchain]
channel = "<version from step 4 of Inputs>"
components = ["rustfmt", "clippy", "rust-analyzer"]
```

Purpose: pin the toolchain for local development. Any `cd` into the repo makes `rustup`'s shims dispatch `cargo`, `rustfmt`, `clippy`, etc. to this exact version. Without this file, local dev uses whatever `rustup default` points at, which drifts from CI's pinned version and causes silent `fmt-check` / clippy failures on push.

The `channel` value MUST be byte-identical to every CI workflow's `toolchain:` input. Two files, one invariant.

### 2b. Enforce sync between `rust-toolchain.toml` and workflow pins

After creating both the workflow files (section 2) and `rust-toolchain.toml` (section 2a), verify the invariant before reporting success:

1. Parse `channel` from `rust-toolchain.toml`
2. For every `.github/workflows/*.yml`, find every `dtolnay/rust-toolchain@master` step and read its `toolchain:` input
3. If any workflow value disagrees with the `rust-toolchain.toml` channel, stop and report the mismatch. Do NOT silently "fix" one side — the user must decide which version is correct.

This check is mandatory and must pass before the command reports success.

### 2c. Migration: existing repo with CI pins but no `rust-toolchain.toml`

If the repo was set up by an older version of this command and has hardcoded `toolchain: X.Y.Z` in CI workflows but no `rust-toolchain.toml`:

1. Read the pinned version from every `dtolnay/rust-toolchain@master` step in `.github/workflows/*.yml`. They SHOULD all agree; if not, stop and report the disagreement.
2. Compare to local `rustc --version`. If they disagree, ask the user which should win. Do not assume.
3. Write `rust-toolchain.toml` with the chosen version (per 2a).
4. If the user chose a version different from what's in CI, update every workflow's `toolchain:` input to match.
5. Run `rustup show` to install/activate the toolchain locally. This may trigger a rustup download — that's expected.
6. Run the invariant check from 2b.

### 3. Add CI section to justfile header

```just
# Run all CI checks (same as GitHub Actions!)
# This is what developers should run before pushing
ci: fmt-check lint test build
```

## Constraints

- `Cargo.lock` MUST be committed to git. For binary crates this is required; for library crates it is still recommended for CI reproducibility. If it is gitignored, remove it from `.gitignore` and commit it. All cargo commands in CI and in the justfile must use `--locked` so that a stale lockfile is a build failure, not a silent re-resolve.
- `rust-toolchain.toml` at the repo root pins the toolchain for local dev. Every CI workflow's `dtolnay/rust-toolchain@master` step pins the same version in its `toolchain:` input. These two sources MUST agree — the command enforces this invariant before reporting success (section 2b)
- Do NOT use `dtolnay/rust-toolchain@stable` — it resolves to different versions on different days, causing CI failures that pass locally. Use `@master` with an explicit version string.
- Do NOT modify Rust source code, Cargo.toml dependencies, or library code
- Do NOT change the project's Rust edition or toolchain version — read what's configured
- Do NOT add release, publishing, benchmarking, or security scanning — those are separate concerns
- If a justfile already exists, preserve existing recipes and add missing ones
- If CI workflows already exist, update them to use `just ci` rather than duplicating build logic

## Final step

Update relevant docs (README.md or ARCHITECTURE.md) to document:
- How to run CI locally: `just ci`
- What CI checks: formatting, clippy (warnings=errors), tests, build

<!-- Reference: refined after a drift incident in navicore/patch-seq where local rustfmt disagreed with CI rustfmt. Local was running unpinned rustup stable; CI was pinned to 1.93.0. Fix: rust-toolchain.toml pins local to match CI, and the command enforces the two stay in sync. dtolnay/rust-toolchain does NOT auto-read the file — both places must be written and verified. -->
