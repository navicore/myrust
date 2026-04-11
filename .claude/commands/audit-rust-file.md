Audit and (with approval) refactor a single Rust file for file-level health: header doc, function length, nesting, redundancy, `unwrap` hygiene, dead code, visibility, and terse docs. Pairs with `/audit-rust-plan`. $ARGUMENTS

## Inputs

1. File path from `$ARGUMENTS`. If empty, read the most recent `tmp/rust-audit-*.md`, pick the first unchecked item (HIGH bucket first), and confirm with the user before proceeding.
2. Read the target file in full — this is an expensive command, full read is the point.
3. Read just enough of the containing crate's `lib.rs` / `mod.rs` / nearest parent module to understand how this file is used. Stop as soon as you know the file's role — don't wander.
4. Run `just lint` if a justfile exists, else `cargo clippy -p <crate> --locked -- -D warnings`. Capture any warnings that land inside this file; treat them as pre-existing findings.

## Audit — report, do not fix

Produce a numbered findings list. Each entry: `file:line` · severity (`nit` | `smell` | `fix`) · one line describing the smell (not the fix).

### A. File header & role
1. Missing `//!` module-level doc at the top stating the file's role in one sentence. (File comment is required; item comments are only where the name alone isn't enough.)
2. If the file is purely `pub use` re-exports, note that and skip B–G.

### B. Size & shape
3. File >500 L → `smell`; >1000 L → `fix` with a suggested split axis.
4. Any fn >80 L → `smell`; >150 L → `fix`. Report the exact line range.
5. Any fn body with max indent depth >4 → `smell`; >6 → `fix`.
6. Any `match` with >20 arms → `smell` (suggest dispatch table or helper).
7. Struct with >10 fields or enum with >15 variants → `smell`.
8. Fn with >5 parameters → `smell` (suggest arg struct or builder).

### C. Docs hygiene — terse, not absent
9. Public items (`pub fn` / `pub struct` / `pub enum` / `pub trait` / `pub const`) missing `///` docs ONLY when the name alone doesn't convey intent. Do not flag `pub fn len() -> usize` for missing docs. Do flag `pub fn process()`.
10. `//` or `///` that only restates the item name or the next line of code → `fix` (delete; the name is the doc).
11. Commented-out code blocks → `fix` (delete; git remembers).
12. `TODO` / `FIXME` / `XXX` — list them but do NOT auto-remove; these encode context.

### D. Redundancy within the file
13. Repeated 3+ line blocks → suggest a local helper.
14. Parallel match arms doing nearly-identical work → suggest data-driven dispatch.
15. Repeated `.map_err(|e| ...)` boilerplate → suggest `From` impl or `?`.
16. Parallel struct/enum shapes → suggest generic or shared type.

### E. Rust idioms (file-scoped only)
17. `.clone()` where a borrow would suffice, especially in loops.
18. `.unwrap()` / `.expect(` outside `#[cfg(test)]` where a real error would be appropriate.
19. `String` parameter where `&str` or `Cow<str>` would do.
20. `Vec<T>` parameter where `&[T]` would do.
21. `bool` parameter on a function with non-obvious call sites → suggest a two-variant enum.
22. `if let Some(x) = y { ... } else { return ...; }` → suggest `let ... else`.
23. Deep `if let` / `match` chains on `Option` / `Result` that could be `?`.
24. Manual `Default` impl where `#[derive(Default)]` would do.
25. `#[allow(dead_code)]` hiding items that really are unused (confirm with clippy before flagging `fix`).
26. `use super::*;` outside `#[cfg(test)]`.

### F. Visibility discipline
27. `pub` items used only within the crate → suggest `pub(crate)`.
28. `pub(crate)` items used only within the module → suggest private.
29. Deep re-export paths that could be flattened at the module root.

### G. Layout
30. Multiple `impl` blocks for the same type with no reason (cfg gates, trait bounds differing, etc.) → suggest merge.
31. Items in a surprising order: convention is `use` → types → inherent `impl` → trait `impl` → free fns → `#[cfg(test)] mod tests`.
32. `#[cfg(test)] mod tests` not at the bottom.

### H. Clippy echo
33. Any clippy warning from step 4 that points inside this file, quoted verbatim with its lint name.

## Plan & approval

After findings, propose a **refactor plan**: an ordered list of concrete edits, each one `file:line` + one sentence. Group related edits (e.g. "extract helper `parse_header`" groups findings 13, 14, 16). Ask the user which groups to apply. **Do not edit before approval.**

When the user approves only some groups, apply only those. Don't sneak in extras.

## Applying

1. Apply edits group-by-group using `Edit`.
2. Stay in this single file. If a finding needs cross-file surgery (move a fn to a sibling module, change a type used elsewhere), record it under the checklist entry as a deferred note — do NOT execute it here.
3. Re-run `just ci` (or at minimum `just fmt lint test`). If anything fails, diagnose and fix the code — do not weaken tests, do not suppress clippy.
4. Update `tmp/rust-audit-*.md`:
   - Tick the file's checkbox.
   - Append one line underneath: `  - ✓ <short summary of what changed, e.g. "split parse_header out of parse, removed 3 unwraps, added file-header doc">`.
   - If there are deferred cross-file items, list them as further sub-bullets prefixed with `↗`.

## Constraints

- **One file only.** Cross-file refactors are deferred, not silently done.
- **Never modify tests** without explicit user approval — this repo's global rule. If a refactor would require a test change, flag it as a finding and stop.
- Do NOT bump crate versions or edit `Cargo.toml` / `Cargo.lock`.
- Do NOT add or remove dependencies.
- Do NOT introduce abstractions beyond what's in the approved plan — no speculative helpers, traits, or generics.
- Do NOT add `#[must_use]`, `#[inline]`, derive macros, or lint attributes the user didn't agree to. Flag them as findings instead.
- Do NOT delete `TODO` / `FIXME` / `#[allow(...)]` without confirming each one — they often encode context you can't see.
- Do NOT change public API shape (names, signatures, visibility) unless the user explicitly approved that group of findings.
- If clippy is unhappy after the refactor, fix the code — not the lint config, not `#[allow]`.
- Keep the file under ~200 lines where practical (user preference); if a split is needed, propose it as a deferred cross-file finding rather than executing.
