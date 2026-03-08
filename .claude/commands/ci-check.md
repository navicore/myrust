(myrust) Run the same checks CI will run, using `just ci`.

Before reporting that work is done, run `just ci` and verify it passes. If it
fails, fix the issue — don't suggest the user fix it later. CI failures on push
waste time and break flow.

Do NOT run individual cargo commands. The `just ci` recipe defines the exact
sequence and flags that GHA will use.

When clippy warns, fix the code. Do NOT add `#[allow(...)]` or `#[cfg_attr(...)]`
annotations to suppress warnings without discussing it first. The fix is almost
always better code, not a silenced lint.
