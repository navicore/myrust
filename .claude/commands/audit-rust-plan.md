Scan a Rust source tree and produce a file-level audit checklist — a list of `.rs` files ranked by cheap size/complexity signals so you can work through them one at a time with `/audit-rust-file`. The checklist is durable and resumable across sessions. $ARGUMENTS

## Inputs

1. Scope path from `$ARGUMENTS` — a crate dir, module dir, or relative path under a workspace. If empty, ask the user which area to audit. Do NOT default to the whole workspace: file-level audit is expensive and needs a tight scope.
2. Read the workspace `Cargo.toml` to confirm this is a Rust project and learn the crate layout.
3. Check `.gitignore`. The checklist will be written under `tmp/`; if `tmp/` isn't gitignored, warn before writing.

## Steps

### 1. Enumerate candidate files

Find `.rs` files under the scope, excluding:
- `target/`, `tmp/`, `.git/`
- `tests/` (integration tests — different concerns, different skill)
- `benches/`, `examples/`
- Files containing `// @generated` or names ending in `.generated.rs`
- `build.rs`

### 2. Compute cheap per-file signals

For each file, collect only structural signals — no LLM reading:
- **Lines**: `wc -l`
- **Function count**: Grep for `^\s*(pub(\(.*\))?\s+)?(async\s+)?fn\s` (count)
- **Max indent depth**: longest leading-whitespace run ÷ 4 (proxy for nesting)
- **Longest fn proxy**: largest gap in line numbers between consecutive `fn ` starts (rough; exact AST is out of scope)
- **TODO/FIXME/XXX** count
- **`unwrap()` / `expect(` count outside `#[cfg(test)]`** (Grep with `-v` on the cfg-test block is imperfect; approximate by excluding files under `tests/` mods — note in findings)
- **Public-surface count**: lines starting with `pub fn `, `pub struct `, `pub enum `, `pub trait `

### 3. Rank

Bucket each file:
- **HIGH**: lines >500, OR longest-fn proxy >120, OR max indent depth ≥6
- **MEDIUM**: lines 300–500, OR depth 5, OR >20 functions
- **LOW**: everything else
- **skip-likely**: <80 lines AND mostly `pub use` / type aliases / trivial re-exports

Sort HIGH → MEDIUM → LOW → skip-likely, and within each bucket by lines descending.

### 4. Write the checklist

Write `tmp/rust-audit-<slug>.md` where `<slug>` is the last path component lowercased with non-alphanumerics replaced by `-`. Create `tmp/` if missing.

Format:

```markdown
# Rust file audit: <scope path>

Baseline: <YYYY-MM-DD>. <N> files, <T> lines total. Scope: `<path>`.

Audit criteria: file header doc, function length, nesting depth, redundant
comments, dead code, `unwrap` hygiene, visibility, layout. Run
`/audit-rust-file <path>` on any unchecked item, or with no arg to pick the
next HIGH item.

## HIGH

- [ ] `crates/compiler/src/parser.rs` — 892 L · 34 fns · depth 7 · 4 TODO · 12 unwrap
- [ ] ...

## MEDIUM

- [ ] ...

## LOW

- [ ] ...

## skip-likely

- [ ] `crates/core/src/lib.rs` — 42 L · re-exports only
```

Leave room under each item for `/audit-rust-file` to append a one-line result after it ticks the box.

### 5. Report

Print to the user:
- Checklist path.
- Bucket counts (HIGH / MEDIUM / LOW / skip-likely).
- The three biggest offenders by lines, so the next step is obvious.

## Constraints

- Do NOT open files and reason about their contents. This command is structural scanning only — that's the entire point of splitting plan from audit.
- Do NOT modify any source file.
- Do NOT commit the checklist.
- If the scope path is ambiguous (matches multiple crates, or doesn't exist), stop and ask.
- If a checklist for the same slug already exists with unchecked items, ask before overwriting — the user may be mid-audit. Offer to rename the new one with a date suffix.
- Do NOT run `cargo build`, `cargo check`, or any tool that compiles code — this command must be fast and cheap.
