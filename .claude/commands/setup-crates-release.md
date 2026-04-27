(myrust) Set up a GitHub Actions release workflow that publishes Rust crates to crates.io when a GitHub release is tagged. Auto-updates Cargo.toml versions from the tag. $ARGUMENTS

## Inputs

1. Read the root `Cargo.toml` and **determine the project shape**:
   - **Single-crate**: has `[package]` at the root, no `[workspace]` section. One crate to publish.
   - **Workspace**: has `[workspace]` with `members = [...]`. May also have `[workspace.package]` for shared version/metadata. Multiple crates to publish.
2. If workspace: read each member's `Cargo.toml` to map inter-crate dependencies (which crates reference each other with `path` + `version`) and determine publish order (leaf dependencies first, dependents last).
3. If single-crate: no dependency ordering needed — there's only one crate.
4. Read `.github/workflows/` to check for existing release workflows.

The rest of this command has two tracks — **single-crate** and **workspace**. Apply only the track that matches the project. Do not add workspace machinery to a single-crate project or vice versa.

## Steps

### 1. Create release workflow: `.github/workflows/release.yml`

Triggers on: `release: types: [created]` with condition `startsWith(github.ref, 'refs/tags/v')`

**a. Extract version from tag:**
```yaml
- name: Extract version from tag
  id: version
  run: |
    VERSION="${GITHUB_REF_NAME#v}"
    echo "version=$VERSION" >> $GITHUB_OUTPUT
```

**b. Update Cargo.toml version(s):**

Use `awk` (not `sed`) so the replacement is scoped to the correct TOML section and doesn't corrupt dependency version strings. Target the section that owns the version for the project shape:

- **Single-crate**: target `[package]`
```bash
awk -v new_version="$VERSION" '
  /^\[package\]/ { in_section=1 }
  /^\[/ && !/^\[package\]/ { in_section=0 }
  in_section && /^version = / {
    print "version = \"" new_version "\""
    in_section=0
    next
  }
  { print }
' Cargo.toml > Cargo.toml.tmp && mv Cargo.toml.tmp Cargo.toml
```

- **Workspace**: target `[workspace.package]`
```bash
awk -v new_version="$VERSION" '
  /^\[workspace\.package\]/ { in_section=1 }
  /^\[/ && !/^\[workspace\.package\]/ { in_section=0 }
  in_section && /^version = / {
    print "version = \"" new_version "\""
    in_section=0
    next
  }
  { print }
' Cargo.toml > Cargo.toml.tmp && mv Cargo.toml.tmp Cargo.toml
```

- **Workspace only**: also update pinned version strings (`version = "=X.Y.Z"`) in inter-crate dependencies across member `Cargo.toml` files using similar section-scoped awk scripts. Skip this step entirely for single-crate projects.
- Run `cargo generate-lockfile` after changes.
- Commit as `"chore: bump version to X.Y.Z"` using `github-actions[bot]` identity and push to main.

**c. Checkout must use a PAT** (not `GITHUB_TOKEN`) to allow pushing back:
```yaml
- uses: actions/checkout@v5
  with:
    token: ${{ secrets.PAT }}
    ref: main
```

**c2. Pin the Rust toolchain — same version as CI:**

If `rust-toolchain.toml` exists at the repo root (i.e. `setup-rust-ci` has been run), the release workflow MUST pin the same version explicitly. Use `dtolnay/rust-toolchain@master` with an explicit `toolchain:` input — NOT `@stable`.

```yaml
# Pinned to match `rust-toolchain.toml` and `.github/workflows/ci-*.yml`.
# All three values MUST agree — see `setup-rust-ci` invariant.
- uses: dtolnay/rust-toolchain@master
  with:
    toolchain: <same version as rust-toolchain.toml channel>
```

Why: `release.yml` runs `cargo generate-lockfile` and `cargo publish`. If it uses `@stable`, it can resolve to a different rustc than CI tested with, producing a `Cargo.lock` that CI never saw — which is exactly the drift `setup-rust-ci` exists to prevent. Pin to the same version, and `release.yml` becomes part of the same invariant `setup-rust-ci` enforces (see its section 2b: "For every `.github/workflows/*.yml`...").

If `rust-toolchain.toml` does NOT exist, recommend the user run `setup-rust-ci` first — leaving the release workflow on `@stable` while CI is also unpinned just defers the drift problem.

**d. Verify versions match** before publishing:
- **Single-crate**: confirm `[package].version` matches the tag.
- **Workspace**: each member should use `version.workspace = true`, and inter-crate dependency pins should match the workspace version.

**e. Publish to crates.io:**

Common flags for both shapes:
- Use `--no-verify` (already tested by CI)
- Handle re-publishes gracefully: `|| echo "::warning::publish failed (may already exist)"`
- Use `CRATES_IO_TOKEN` secret

**Single-crate** — one publish, no ordering, no sleep:
```yaml
- run: cargo publish --no-verify
```

**Workspace** — publish in dependency order (leaf crates first, then their dependents), with a **120 second** sleep between publishes to let crates.io index each one before the next depends on it.

Example for a workspace with `core → runtime → compiler → lsp`:
```yaml
- run: cargo publish -p my-core --no-verify
- run: sleep 120
- run: cargo publish -p my-runtime --no-verify
- run: sleep 120
- run: cargo publish -p my-compiler --no-verify
```

**f. (Optional) Build release binaries:**
If the project produces binaries, add jobs after publish:
- Linux: build with `x86_64-unknown-linux-musl` for static linking
- macOS: build on `macos-latest` for ARM64
- Upload artifacts with `softprops/action-gh-release@v1`

### 2. Required GitHub secrets

Tell the user to configure in repo Settings → Secrets:
- `PAT`: Personal access token with `contents: write` scope (for pushing version bumps back to main)
- `CRATES_IO_TOKEN`: API token from https://crates.io/settings/tokens

### 3. Verify version setup

- **Single-crate**: confirm the root `[package].version` is the single source of truth. Nothing else to do.
- **Workspace**: check that each member's `Cargo.toml` uses `version.workspace = true` under `[package]`. If any crate hardcodes a version, update it to use the workspace version. This ensures a single version bump in the workspace root propagates everywhere.

## Constraints

- Do NOT modify Rust source code or library logic
- Do NOT set up CI (that's `setup-rust-ci`) — only the release/publish workflow
- The awk version-update must be scoped to the correct TOML section to avoid corrupting dependency version strings
- Inter-crate version pins (`version = "=X.Y.Z"`) and publish ordering only apply to workspaces — skip both for single-crate projects
- For workspaces, publish order must respect the dependency graph — publishing a crate before its dependencies are indexed will fail
- If `rust-toolchain.toml` exists, `release.yml` MUST use `dtolnay/rust-toolchain@master` with an explicit `toolchain:` input matching the file's `channel` value. Never use `@stable` here — it bypasses the toolchain pin that CI relies on and reintroduces drift between what CI tests and what release publishes.

## Final step

Update relevant docs (README.md or ARCHITECTURE.md) to document:
- How to release: create a GitHub release with tag `vX.Y.Z`
- Required secrets: `PAT` and `CRATES_IO_TOKEN`
- What happens automatically: version bump, publish, (optional) binary builds

<!-- Reference: pattern extracted from navicore/patch-seq -->

<!-- Reference: section 1c2 (toolchain pin) was added after navicore/patch-rexx — release.yml had been generated with `@stable` while CI and rust-toolchain.toml were pinned to a specific version, recreating the drift problem `setup-rust-ci` was designed to eliminate. The pin in release.yml must be part of the same invariant. -->
