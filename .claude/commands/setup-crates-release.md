(myrust) Set up a GitHub Actions release workflow that publishes Rust crates to crates.io when a GitHub release is tagged. Auto-updates Cargo.toml versions from the tag. $ARGUMENTS

## Inputs

1. Read `Cargo.toml` to find: workspace members (if any), current version, `[workspace.package]` vs `[package]` structure
2. Read all workspace member `Cargo.toml` files to map inter-crate dependencies (which crates reference each other with `path` + `version`)
3. Read `.github/workflows/` to check for existing release workflows
4. Determine the crate publish order (leaf dependencies first, dependents last)

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
- Read current version from `[workspace.package]` (or `[package]` for single-crate)
- If it differs from tag, update using `awk` (not `sed` — awk handles section-scoped replacement):
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
- For workspaces: also update pinned version strings (`version = "=X.Y.Z"`) in inter-crate dependencies using similar awk scripts scoped to the correct lines
- Run `cargo generate-lockfile` after changes
- Commit as `"chore: bump version to X.Y.Z"` using `github-actions[bot]` identity and push to main

**c. Checkout must use a PAT** (not `GITHUB_TOKEN`) to allow pushing back:
```yaml
- uses: actions/checkout@v5
  with:
    token: ${{ secrets.PAT }}
    ref: main
```

**d. Verify versions match** before publishing:
- Each workspace member should use `version.workspace = true`
- Inter-crate dependency pins should match workspace version

**e. Publish to crates.io in dependency order:**
- Leaf crates first (no internal dependencies), then their dependents
- Wait **120 seconds** between publishes (crates.io indexing delay)
- Use `--no-verify` (already tested by CI)
- Handle re-publishes gracefully: `|| echo "::warning::publish failed (may already exist)"`
- Use `CRATES_IO_TOKEN` secret

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

### 3. Verify workspace version setup

Check that each crate's `Cargo.toml` uses `version.workspace = true` under `[package]`. If any crate hardcodes a version, update it to use the workspace version. This ensures a single version bump in the workspace root propagates everywhere.

## Constraints

- Do NOT modify Rust source code or library logic
- Do NOT set up CI (that's `setup-rust-ci`) — only the release/publish workflow
- The awk version-update must be scoped to the correct TOML section to avoid corrupting dependency version strings
- Inter-crate version pins (`version = "=X.Y.Z"`) only apply to workspace members that reference each other with both `path` and `version` — read the actual files to determine which ones
- The publish order must respect the dependency graph — publishing a crate before its dependencies are indexed will fail

## Final step

Update relevant docs (README.md or ARCHITECTURE.md) to document:
- How to release: create a GitHub release with tag `vX.Y.Z`
- Required secrets: `PAT` and `CRATES_IO_TOKEN`
- What happens automatically: version bump, publish, (optional) binary builds

<!-- Reference: pattern extracted from navicore/patch-seq -->
