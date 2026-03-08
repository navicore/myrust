(myrust) Verify the justfile is the single source of truth for build operations.

1. Read the `justfile` and all `.github/workflows/*.yml` files
2. Check that:
   - GHA workflows call `just` recipes — never raw `cargo` commands (except toolchain setup)
   - Every CI step maps to a `just` recipe
   - No build/test/lint logic is duplicated between justfile and workflows
   - Clippy flags, test scopes, and build targets are identical in both
3. Report any drift between local `just` recipes and CI workflows

If $ARGUMENTS is provided, focus on: $ARGUMENTS
