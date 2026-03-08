(myrust) Run the appropriate just recipe. Do NOT use raw cargo commands.

This project uses `just` as the build system. The justfile is the source of truth
for all build, test, lint, and CI operations. Raw `cargo` commands bypass project
conventions and may miss flags, features, or configuration.

- Build: `just build`
- Test: `just test`
- Lint: `just lint`
- Format: `just fmt`
- Full CI check: `just ci`
- See all recipes: `just --list`

Run: just $ARGUMENTS
