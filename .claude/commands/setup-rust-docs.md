(myrust) Set up GitHub Actions to build and deploy a documentation site for a Rust project to GitHub Pages. Picks between a plain rustdoc site and an mdBook site at invocation time. $ARGUMENTS

## Inputs

1. Read `Cargo.toml` to determine project shape (single crate vs workspace) and list crate-root paths.
2. Read `.github/workflows/` to check for existing `docs.yml` or similar — do not overwrite.
3. Read `justfile` if it exists — append recipes rather than rewriting.
4. Read the repo root for hints about which shape the user already leans toward:
   - `book.toml` present, or `docs/` dir with multiple `.md` files + a `SUMMARY.md` → suggest **mdbook**.
   - Neither → suggest **rustdoc**.
5. Read `README.md` at the repo root — the README integration is part of the skill's value; refuse to proceed if none exists (tell the user to `git add` one first).
6. Verify the repo has at least one pushed commit on the default branch `main`. GitHub Pages deploys don't fire on initial-push-less repos.

## Step 0. Choose the shape

Prompt the user with a single question:

> "Generate a `rustdoc` site (pure `cargo doc`, minimal) or an `mdbook` site (prose-first, supports guides/examples)?"

Default the suggestion based on the input-4 heuristic. Accept one answer and do not ask again. Everything below is scoped to the chosen shape.

---

## Rustdoc track

### R1. Wire the README into each crate root

For every crate root file in the workspace (`src/lib.rs`, or `src/main.rs` for bin-only crates without a lib), insert at the top:

```rust
#![doc = include_str!("../README.md")]
```

For bin-only crates without a `README.md` adjacent to `Cargo.toml`, fall back to the workspace-root README with the appropriate relative path. Skip files that already have a `doc = include_str!` line pointing at the README.

### R2. Turn on strict doc lints

Add a `[lints.rust]` block to the root `Cargo.toml` (or `[workspace.lints.rust]` for workspaces). If the table exists, merge — don't replace:

```toml
[lints.rust]
missing_docs = "warn"

[lints.rustdoc]
broken_intra_doc_links = "warn"
private_intra_doc_links = "warn"
```

Tell the user in the exit message that `cargo doc` will now warn on undocumented `pub` items. This is the intended side effect, not a regression.

### R3. Workflow: `.github/workflows/docs.yml`

```yaml
name: Deploy Documentation

on:
  push:
    branches: [main]
    paths:
      - 'crates/**'       # adjust paths glob to match the project
      - 'src/**'
      - 'Cargo.toml'
      - 'Cargo.lock'
      - 'README.md'
      - '.github/workflows/docs.yml'
  workflow_dispatch:

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: "pages"
  cancel-in-progress: false

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@master
        with:
          toolchain: <pin from rust-toolchain.toml>
      - run: cargo doc --locked --no-deps --workspace
      - name: Add index redirect
        run: |
          # Single-crate: put a meta-refresh at target/doc/index.html pointing
          # at the one crate. Workspace: pick the "main" crate (workspace
          # default-members[0] or the crate named like the repo).
          CRATE="<name-derived-from-Cargo.toml>"
          echo "<meta http-equiv=refresh content=\"0; url=${CRATE}/index.html\">" > target/doc/index.html
      - uses: actions/configure-pages@v4
      - uses: actions/upload-pages-artifact@v3
        with:
          path: ./target/doc

  deploy:
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    needs: build
    steps:
      - uses: actions/deploy-pages@v4
        id: deployment
```

Pin the toolchain to whatever `rust-toolchain.toml` already contains (same invariant as `setup-rust-ci`). If there's no `rust-toolchain.toml`, tell the user to run `setup-rust-ci` first — don't guess.

### R4. Justfile recipes

Append to `justfile` if present (create if not):

```just
# Build rustdoc locally — mirrors what CI does
docs:
    cargo doc --locked --no-deps --workspace --open

# Build without opening (for CI-equivalent checks)
docs-check:
    cargo doc --locked --no-deps --workspace
```

---

## mdBook track

### M1. `book.toml`

Write `book.toml` at the repo root. Do not overwrite if present.

```toml
[book]
title = "<repo name titled>"
authors = ["<git config user.name>"]
description = "Documentation for <repo name>"
src = "docs"
language = "en"

[build]
build-dir = "book"

[output.html]
default-theme = "rust"
preferred-dark-theme = "ayu"
git-repository-url = "<https url inferred from `git remote get-url origin`>"
edit-url-template = "<git-repository-url>/edit/main/docs/{path}"
site-url = "/<repo name>/"

[output.html.fold]
enable = true
level = 1

[output.html.search]
enable = true
limit-results = 30
boost-hierarchy = 2
```

### M2. `docs/SUMMARY.md`

If `docs/` already has `.md` files, build a best-effort SUMMARY that lists `README.md` as the intro plus every top-level `.md` in alphabetical order under a "Reference" heading. If `docs/` is empty, seed with:

```markdown
# Summary

[Introduction](README.md)

# Getting Started

- [Overview](README.md)
```

Do not overwrite an existing `docs/SUMMARY.md`.

### M3. README integration

mdBook renders `docs/README.md` as the introduction. If the repo's root `README.md` uses markdown constructs mdBook mangles (e.g. `##` headers appearing in the sidebar), write a generator script `scripts/generate-docs.sh`:

```bash
#!/bin/bash
# Regenerate docs/README.md from the repo-root README.md.
# Convert `##` headers to bold so they don't clutter the mdBook sidebar.
set -e
sed 's/^## \(.*\)/**\1**/' README.md > docs/README.md
```

Add to `.gitignore`:

```gitignore
/book/
```

If the repo has an `examples/` tree with per-folder README files, extend the generator to stitch them into `docs/EXAMPLES.md` with category-based header-level adjustment. Use the header-adjust pattern the user can crib from patch-seq's `scripts/generate-examples-docs.sh`.

### M4. Workflow: `.github/workflows/docs.yml`

```yaml
name: Deploy Documentation

on:
  push:
    branches: [main]
    paths:
      - 'docs/**'
      - 'examples/**/README.md'
      - 'book.toml'
      - 'README.md'
      - '.github/workflows/docs.yml'
      - 'scripts/generate-docs.sh'
  workflow_dispatch:

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: "pages"
  cancel-in-progress: false

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: peaceiris/actions-mdbook@v2
        with:
          mdbook-version: 'latest'
      - name: Generate docs from README/examples
        run: ./scripts/generate-docs.sh
      - run: mdbook build
      - uses: actions/configure-pages@v4
      - uses: actions/upload-pages-artifact@v3
        with:
          path: ./book

  deploy:
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    needs: build
    steps:
      - uses: actions/deploy-pages@v4
        id: deployment
```

### M5. Justfile recipes

Append to `justfile` (create if not):

```just
# Regenerate docs/README.md (and examples if applicable) from source
gen-docs:
    ./scripts/generate-docs.sh

# Build the mdbook site into ./book/
build-docs: gen-docs
    mdbook build

# Serve locally with live reload
docs:
    just gen-docs
    mdbook serve --open
```

---

## Final step (both tracks)

### F1. Configure GitHub Pages source

Tell the user to do this manually (the skill can't):

> **GitHub Pages setup (one-time, per-repo):** Go to repo Settings → Pages → Source, select **GitHub Actions**. The first push after this setting change will trigger a deploy.

### F2. Update project docs

Append to `README.md` (or `ARCHITECTURE.md`) under a new "Documentation" section:

- **Rustdoc track:** "Rendered docs are published to `https://<user>.github.io/<repo>/`. Build locally with `just docs`. The root `README.md` is included as the crate-level docs."
- **mdBook track:** "Rendered docs are published to `https://<user>.github.io/<repo>/`. Build locally with `just docs` (serve) or `just build-docs` (one-shot). The site pulls content from `docs/` plus a generated copy of `README.md`."

### F3. Exit message

Report clearly:
- Which shape was chosen.
- Which files were written and which were skipped (because they already existed).
- The GitHub Pages manual-configure reminder from F1.
- The "strict doc lints now on" warning from R2 (rustdoc track only).
- Next step: commit + push to `main`; first deploy fires automatically.

## Constraints

- **Do not overwrite existing files.** If `.github/workflows/docs.yml`, `book.toml`, `docs/SUMMARY.md`, `scripts/generate-docs.sh`, or any targeted `#![doc = include_str!(...)]` line already exist, leave them alone and report in the exit message that they were skipped. Silent merges hide bugs.
- **Do not run `setup-rust-ci` or modify `ci.yml`.** Docs is its own workflow. The two skills are composable, not overlapping.
- **Do not bump crate versions, add dependencies, or edit `Cargo.lock`.** Only the `[lints]` additions in R2 touch `Cargo.toml`.
- **Do not create a `rust-toolchain.toml`** — that's `setup-rust-ci`'s job. If none exists, stop and tell the user to run `setup-rust-ci` first.
- **Do not pick both shapes.** If the user asks for "both," explain that rustdoc and mdbook would race on the same Pages deploy slot and force them to choose one.
- **One Pages deploy per repo.** If another workflow already targets Pages (look for `actions/deploy-pages@v4` or `actions/configure-pages@v4` in any `.github/workflows/*.yml`), stop and ask the user whether to replace it; do not add a second.

## Final constraint

This skill is a one-shot setup. It does not have update/refresh/migrate modes. If the user wants to change shape later (e.g. rustdoc → mdbook), they delete the generated files by hand and re-run the skill.

<!-- Reference: pattern extracted from navicore/patch-seq (mdbook track reproduces docs.yml + book.toml + scripts/generate-examples-docs.sh). Design doc: patch-seq/docs/design/MYRUST_DOC_SKILLS.md -->
